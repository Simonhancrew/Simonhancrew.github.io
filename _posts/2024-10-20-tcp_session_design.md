---
title: tcp session design
date: 2024-10-20 17:03:10 +0800
categories: [Blogging, muduo]
tags: [writing]
---

tcp的server跟client的逻辑不尽相似, 但是最下层的都是tcp connection

### tcp connection

先简单看一下构造

```cpp
TcpConnection::TcpConnection(EventLoop* loop,
                             const string& nameArg,
                             int sockfd,
                             const InetAddress& localAddr,
                             const InetAddress& peerAddr)
  : loop_(CHECK_NOTNULL(loop)),
    name_(nameArg),
    state_(kConnecting),
    reading_(true),
    socket_(new Socket(sockfd)),
    channel_(new Channel(loop, sockfd)),
    localAddr_(localAddr),
    peerAddr_(peerAddr),
    highWaterMark_(64*1024*1024)
{
  channel_->setReadCallback(
      std::bind(&TcpConnection::handleRead, this, _1));
  channel_->setWriteCallback(
      std::bind(&TcpConnection::handleWrite, this));
  channel_->setCloseCallback(
      std::bind(&TcpConnection::handleClose, this));
  channel_->setErrorCallback(
      std::bind(&TcpConnection::handleError, this));
  LOG_DEBUG << "TcpConnection::ctor[" <<  name_ << "] at " << this
            << " fd=" << sockfd;
  socket_->setKeepAlive(true);
}
```

比较容易看到的是这里面其实是有一个状态机state的, 第二个点就是这里有一个highWaterMark, 先看下这个是干嘛的

```cpp
enum StateE { kDisconnected, kConnecting, kConnected, kDisconnecting };
```

可以看到第一处跟watermark相关的代码在send里

```cpp
void TcpConnection::sendInLoop(const void* data, size_t len)
{
  loop_->assertInLoopThread();
  ssize_t nwrote = 0;
  size_t remaining = len;
  bool faultError = false;
  if (state_ == kDisconnected)
  {
    LOG_WARN << "disconnected, give up writing";
    return;
  }
  // if no thing in output queue, try writing directly
  if (!channel_->isWriting() && outputBuffer_.readableBytes() == 0)
  {
    nwrote = sockets::write(channel_->fd(), data, len);
    if (nwrote >= 0)
    {
      remaining = len - nwrote;
      if (remaining == 0 && writeCompleteCallback_)
      {
        loop_->queueInLoop(std::bind(writeCompleteCallback_, shared_from_this()));
      }
    }
    else // nwrote < 0
    {
      nwrote = 0;
      if (errno != EWOULDBLOCK)
      {
        LOG_SYSERR << "TcpConnection::sendInLoop";
        if (errno == EPIPE || errno == ECONNRESET) // FIXME: any others?
        {
          faultError = true;
        }
      }
    }
  }

  assert(remaining <= len);
  if (!faultError && remaining > 0)
  {
    size_t oldLen = outputBuffer_.readableBytes();
    if (oldLen + remaining >= highWaterMark_
        && oldLen < highWaterMark_
        && highWaterMarkCallback_)
    {
      loop_->queueInLoop(std::bind(highWaterMarkCallback_, shared_from_this(), oldLen + remaining));
    }
    outputBuffer_.append(static_cast<const char*>(data)+nwrote, remaining);
    if (!channel_->isWriting())
    {
      channel_->enableWriting();
    }
  }
}
```

这里先判断socket是否可写, 可写的话直接利用wirte, 并且判断remaining的数据是否写完, 写完的话需要考虑complete setReadCallback

随后在有remaing数据没写完的情况下, 看看buffer里缓冲的数据 + remaining的数据跟上次比是不是超过了highWaterMark, 如果超过了, 需要调用highWaterMarkCallback

否则就直接append到buffer里, 并且enableWriting, 这样在下次能写的时候就会调用handleWrite

```cpp
void TcpConnection::handleWrite()
{
  loop_->assertInLoopThread();
  if (channel_->isWriting())
  {
    ssize_t n = sockets::write(channel_->fd(),
                               outputBuffer_.peek(),
                               outputBuffer_.readableBytes());
    if (n > 0)
    {
      outputBuffer_.retrieve(n);
      if (outputBuffer_.readableBytes() == 0)
      {
        channel_->disableWriting();
        if (writeCompleteCallback_)
        {
          loop_->queueInLoop(std::bind(writeCompleteCallback_, shared_from_this()));
        }
        if (state_ == kDisconnecting)
        {
          shutdownInLoop();
        }
      }
    }
    else
    {
      LOG_SYSERR << "TcpConnection::handleWrite";
      // if (state_ == kDisconnecting)
      // {
      //   shutdownInLoop();
      // }
    }
  }
  else
  {
    LOG_TRACE << "Connection fd = " << channel_->fd()
              << " is down, no more writing";
  }
}
```

这里, 在outputbuffer空了之后需要关闭write的callback, 同时也要处理单向关闭相关的逻辑

这块里, 如果还在写, 依然是关不掉的, 所以之前handleWrite里需要处理这个逻辑

```cpp
void TcpConnection::shutdownInLoop()
{
  loop_->assertInLoopThread();
  if (!channel_->isWriting())
  {
    // we are not writing
    socket_->shutdownWrite();
  }
}
```

#### force close的逻辑

这一块还有一个forece close的逻辑, 这块的shared_from_this是为了防止在第一个callback里tcp connection被析构了。

后面一个callback还在使用的话就会发生错误, 这里用裸指针回调就有这个问题

如果你的项目里不能用shared_ptr, 你就只能配合release来使用了, 还得写个destroy guard来保护这部分在调用栈上是不要析构的

```cpp
void TcpConnection::forceCloseInLoop()
{
  loop_->assertInLoopThread();
  if (state_ == kConnected || state_ == kDisconnecting)
  {
    // as if we received 0 byte in handleRead();
    handleClose();
  }
}

void TcpConnection::handleClose()
{
  loop_->assertInLoopThread();
  LOG_TRACE << "fd = " << channel_->fd() << " state = " << stateToString();
  assert(state_ == kConnected || state_ == kDisconnecting);
  // we don't close fd, leave it to dtor, so we can find leaks easily.
  setState(kDisconnected);
  channel_->disableAll();

  TcpConnectionPtr guardThis(shared_from_this());
  connectionCallback_(guardThis);
  // must be the last line
  closeCallback_(guardThis);
}
```

#### 读的逻辑

首先需要链接已经建立了, channel的很多逻辑都在tie之后才回调到上层

```cpp
void TcpConnection::connectEstablished()
{
  loop_->assertInLoopThread();
  assert(state_ == kConnecting);
  setState(kConnected);
  channel_->tie(shared_from_this());
  channel_->enableReading();

  connectionCallback_(shared_from_this());
}
```

这一块都是buffer读, 具体的分包实在是在上层的callback逻辑里处理的, 所以具体的业务层去拼包

```cpp
void TcpConnection::handleRead(Timestamp receiveTime)
{
  loop_->assertInLoopThread();
  int savedErrno = 0;
  ssize_t n = inputBuffer_.readFd(channel_->fd(), &savedErrno);
  if (n > 0)
  {
    messageCallback_(shared_from_this(), &inputBuffer_, receiveTime);
  }
  else if (n == 0)
  {
    handleClose();
  }
  else
  {
    errno = savedErrno;
    LOG_SYSERR << "TcpConnection::handleRead";
    handleError();
  }
}
```

#### 看一下跟tcp socket有关的几个设置

首先构造的时候设置了keepalive

```cpp
void Socket::setKeepAlive(bool on)
{
  int optval = on ? 1 : 0;
  ::setsockopt(sockfd_, SOL_SOCKET, SO_KEEPALIVE,
               &optval, static_cast<socklen_t>(sizeof optval));
  // FIXME CHECK
}
```

+ SO_KEEPALIVE: 这是一个套接字选项，用于启用或禁用 TCP 保活功能。当这个选项被启用时, TCP 会定期向对方发送探测包，以检查连接是否仍然有效。

但是最好还是在业务层配置自己的心跳，系统级别的会有重试相关的逻辑，最坏情况下重建链路的预期可能会比较长

tcp info相关的信息怎么获取

```cpp
bool Socket::getTcpInfo(struct tcp_info* tcpi) const
{
  socklen_t len = sizeof(*tcpi);
  memZero(tcpi, len);
  return ::getsockopt(sockfd_, SOL_TCP, TCP_INFO, tcpi, &len) == 0;
}

bool Socket::getTcpInfoString(char* buf, int len) const
{
  struct tcp_info tcpi;
  bool ok = getTcpInfo(&tcpi);
  if (ok)
  {
    snprintf(buf, len, "unrecovered=%u "
             "rto=%u ato=%u snd_mss=%u rcv_mss=%u "
             "lost=%u retrans=%u rtt=%u rttvar=%u "
             "sshthresh=%u cwnd=%u total_retrans=%u",
             tcpi.tcpi_retransmits,  // Number of unrecovered [RTO] timeouts
             tcpi.tcpi_rto,          // Retransmit timeout in usec
             tcpi.tcpi_ato,          // Predicted tick of soft clock in usec
             tcpi.tcpi_snd_mss,
             tcpi.tcpi_rcv_mss,
             tcpi.tcpi_lost,         // Lost packets
             tcpi.tcpi_retrans,      // Retransmitted packets out
             tcpi.tcpi_rtt,          // Smoothed round trip time in usec
             tcpi.tcpi_rttvar,       // Medium deviation
             tcpi.tcpi_snd_ssthresh,
             tcpi.tcpi_snd_cwnd,
             tcpi.tcpi_total_retrans);  // Total retransmits for entire connection
  }
  return ok;
}
```

不知道是啥，以后再看吧

最后是一个tcp no delay的设置

```cpp
void Socket::setTcpNoDelay(bool on)
{
  int optval = on ? 1 : 0;
  ::setsockopt(sockfd_, IPPROTO_TCP, TCP_NODELAY,
               &optval, static_cast<socklen_t>(sizeof optval));
  // FIXME CHECK
}
```

TCP_NODELAY 是一个套接字选项，用于控制 TCP 协议的 Nagle 算法。Nagle 算法的主要目的是减少网络中的小数据包数量，从而提高网络效率和吞吐量

具体来讲就两个功效

1. 禁用 Nagle 算法: 当你为一个套接字设置 TCP_NODELAY 选项并将其值设置为 1 时, Nagle 算法将被禁用。这意味着小的数据包将被立即发送，而不需要等待合并成更大的数据包。

2. 降低延迟: 禁用 Nagle 算法可以显著降低数据传输的延迟。这对于需要实时数据传输的应用（如在线游戏、视频会议、即时消息等）尤为重要，因为它允许应用程序快速响应并发送数据，而不必等待合并。
