---
title: tcp client design
date: 2024-10-26 15:40:10 +0800
categories: [Blogging, muduo]
tags: [writing]
---


muduo的tcpclient属于tcp connection + connector的组合使用封装

首先看到构造tcpclient，然后connection，有connected回调之后就可以关注业务逻辑了、

### 构造

```cpp
TcpClient::TcpClient(EventLoop* loop,
                     const InetAddress& serverAddr,
                     const string& nameArg)
  : loop_(CHECK_NOTNULL(loop)),
    connector_(new Connector(loop, serverAddr)),
    name_(nameArg),
    connectionCallback_(defaultConnectionCallback),
    messageCallback_(defaultMessageCallback),
    retry_(false),
    connect_(true),
    nextConnId_(1)
{
  connector_->setNewConnectionCallback(
      std::bind(&TcpClient::newConnection, this, _1));
  // FIXME setConnectFailedCallback
  LOG_INFO << "TcpClient::TcpClient[" << name_
           << "] - connector " << get_pointer(connector_);
}
```

这里有connection id，意味着client可以多次connect不同的地址，底下的connection应该是唯一的。随后看到这里有一个connector

```cpp
Connector::Connector(EventLoop* loop, const InetAddress& serverAddr)
  : loop_(loop),
    serverAddr_(serverAddr),
    connect_(false),
    state_(kDisconnected),
    retryDelayMs_(kInitRetryDelayMs)
{
  LOG_DEBUG << "ctor[" << this << "]";
}
```

这个应该就是单纯辅助tcp建联过程的组件

### connect

随后，tcp调用connect

```cpp
void TcpClient::connect()
{
  // FIXME: check state
  LOG_INFO << "TcpClient::connect[" << name_ << "] - connecting to "
           << connector_->serverAddress().toIpPort();
  connect_ = true;
  connector_->start();
}

void Connector::start()
{
  connect_ = true;
  loop_->runInLoop(std::bind(&Connector::startInLoop, this)); // FIXME: unsafe
}

void Connector::startInLoop()
{
  loop_->assertInLoopThread();
  assert(state_ == kDisconnected);
  if (connect_)
  {
    connect();
  }
  else
  {
    LOG_DEBUG << "do not connect";
  }
}

void Connector::connect()
{
  int sockfd = sockets::createNonblockingOrDie(serverAddr_.family());
  int ret = sockets::connect(sockfd, serverAddr_.getSockAddr());
  int savedErrno = (ret == 0) ? 0 : errno;
  switch (savedErrno)
  {
    case 0:
    case EINPROGRESS:
    case EINTR:
    case EISCONN:
      connecting(sockfd);
      break;

    case EAGAIN:
    case EADDRINUSE:
    case EADDRNOTAVAIL:
    case ECONNREFUSED:
    case ENETUNREACH:
      retry(sockfd);
      break;

    case EACCES:
    case EPERM:
    case EAFNOSUPPORT:
    case EALREADY:
    case EBADF:
    case EFAULT:
    case ENOTSOCK:
      LOG_SYSERR << "connect error in Connector::startInLoop " << savedErrno;
      sockets::close(sockfd);
      break;

    default:
      LOG_SYSERR << "Unexpected error in Connector::startInLoop " << savedErrno;
      sockets::close(sockfd);
      // connectErrorCallback_();
      break;
  }
}
```

connect的流程稍微多点，需要处理socket error的报错，一次看下这些报错的原因

在 C 语言中，`case` 语句通常用于 `switch` 语句中，以处理不同的错误代码。在你提供的代码片段中，这些错误代码是与网络编程相关的，通常是在使用套接字（socket）进行连接时可能遇到的错误。以下是你列出的错误代码的解释：

#### 1. `EINPROGRESS`

- **含义**: 表示连接正在进行中。
- **解释**: 当非阻塞套接字调用 `connect()` 时，如果连接尚未完成，系统会返回此错误。程序应继续检查连接状态。

#### 2. `EINTR`

- **含义**: 系统调用被信号中断。
- **解释**: 这是一个常见的错误，表示在执行系统调用时被信号打断。通常需要重新尝试该操作。

#### 3. `EISCONN`

- **含义**: 套接字已经连接。
- **解释**: 表示尝试连接的套接字已经处于连接状态。这通常是因为套接字之前已经成功连接。

#### 4. `EAGAIN`

- **含义**: 资源暂时不可用。
- **解释**: 在非阻塞模式下，操作无法立即完成，通常意味着需要稍后重试。

#### 5. `EADDRINUSE`

- **含义**: 地址已在使用中。
- **解释**: 表示所请求的地址（IP 地址和端口）已被另一个套接字使用，无法绑定或连接。

#### 6. `EADDRNOTAVAIL`

- **含义**: 地址不可用。
- **解释**: 表示所请求的地址在本地不可用，通常是因为网络配置问题。

#### 7. `ECONNREFUSED`

- **含义**: 连接被拒绝。
- **解释**: 表示目标主机的套接字没有在指定的端口上监听，或者目标主机拒绝了连接。

#### 8. `ENETUNREACH`

- **含义**: 网络不可达。
- **解释**: 表示目标网络不可达，可能是由于网络配置问题或路由错误。

#### 9. `EACCES`

- **含义**: 权限被拒绝。
- **解释**: 表示没有足够的权限来执行请求的操作，通常是因为文件或资源的访问权限设置。

#### 10. `EPERM`

- **含义**: 操作不允许。
- **解释**: 表示操作被系统拒绝，通常是因为权限不足。

#### 11. `EAFNOSUPPORT`

- **含义**: 地址族不支持。
- **解释**: 表示所请求的地址族（如 IPv4 或 IPv6）不被支持。

#### 12. `EALREADY`

- **含义**: 连接请求已经在进行中。
- **解释**: 表示套接字的连接请求已经被发起，不能再次发起连接。

#### 13. `EBADF`

- **含义**: 无效的文件描述符。
- **解释**: 表示传递给系统调用的文件描述符无效，通常是因为套接字未正确创建或已关闭。

### 14. `EFAULT`

- **含义**: 错误的地址。
- **解释**: 表示传递给系统调用的指针指向无效的内存地址。

#### 15. `ENOTSOCK`

- **含义**: 不是一个套接字。
- **解释**: 表示操作的文件描述符不是一个有效的套接字。

### 具体的链接和回调处理

首先看正常状况下的connect

```cpp
void Connector::connecting(int sockfd)
{
  setState(kConnecting);
  assert(!channel_);
  channel_.reset(new Channel(loop_, sockfd));
  channel_->setWriteCallback(
      std::bind(&Connector::handleWrite, this)); // FIXME: unsafe
  channel_->setErrorCallback(
      std::bind(&Connector::handleError, this)); // FIXME: unsafe

  // channel_->tie(shared_from_this()); is not working,
  // as channel_ is not managed by shared_ptr
  channel_->enableWriting();
}

void Connector::handleWrite()
{
  LOG_TRACE << "Connector::handleWrite " << state_;

  if (state_ == kConnecting)
  {
    int sockfd = removeAndResetChannel();
    int err = sockets::getSocketError(sockfd);
    if (err)
    {
      LOG_WARN << "Connector::handleWrite - SO_ERROR = "
               << err << " " << strerror_tl(err);
      retry(sockfd);
    }
    else if (sockets::isSelfConnect(sockfd))
    {
      LOG_WARN << "Connector::handleWrite - Self connect";
      retry(sockfd);
    }
    else
    {
      setState(kConnected);
      if (connect_)
      {
        newConnectionCallback_(sockfd);
      }
      else
      {
        sockets::close(sockfd);
      }
    }
  }
  else
  {
    // what happened?
    assert(state_ == kDisconnected);
  }
}


int Connector::removeAndResetChannel()
{
  channel_->disableAll();
  channel_->remove();
  int sockfd = channel_->fd();
  // Can't reset channel_ here, because we are inside Channel::handleEvent
  loop_->queueInLoop(std::bind(&Connector::resetChannel, this)); // FIXME: unsafe
  return sockfd;
}
```

这里创建socket关联channel，然后在handle readevent的时候放弃这个fd，把这个connector的state设置成connected，然后抛newconnection callback。

这里不能立刻reset channel的原因就是这个callback是从channel的`handleEventWithGuard`里抛出来的，而且没用shared ptr之类的做延迟释放的gurad，且在这个callback之后还调用的channel的成员变量，因此这里其实不能立刻释放，不然就会崩

### tcp client在回调中新建连接

```cpp
void TcpClient::newConnection(int sockfd)
{
  loop_->assertInLoopThread();
  InetAddress peerAddr(sockets::getPeerAddr(sockfd));
  char buf[32];
  snprintf(buf, sizeof buf, ":%s#%d", peerAddr.toIpPort().c_str(), nextConnId_);
  ++nextConnId_;
  string connName = name_ + buf;

  InetAddress localAddr(sockets::getLocalAddr(sockfd));
  // FIXME poll with zero timeout to double confirm the new connection
  // FIXME use make_shared if necessary
  TcpConnectionPtr conn(new TcpConnection(loop_,
                                          connName,
                                          sockfd,
                                          localAddr,
                                          peerAddr));

  conn->setConnectionCallback(connectionCallback_);
  conn->setMessageCallback(messageCallback_);
  conn->setWriteCompleteCallback(writeCompleteCallback_);
  conn->setCloseCallback(
      std::bind(&TcpClient::removeConnection, this, _1)); // FIXME: unsafe
  {
    MutexLockGuard lock(mutex_);
    connection_ = conn;
  }
  conn->connectEstablished();
}
```

随后就是connection相关的代码了。

### connector的rery处理

就是存在一个2倍数的退避，最大delay是30s

```cpp
void Connector::retry(int sockfd)
{
  sockets::close(sockfd);
  setState(kDisconnected);
  if (connect_)
  {
    LOG_INFO << "Connector::retry - Retry connecting to " << serverAddr_.toIpPort()
             << " in " << retryDelayMs_ << " milliseconds. ";
    loop_->runAfter(retryDelayMs_/1000.0,
                    std::bind(&Connector::startInLoop, shared_from_this()));
    retryDelayMs_ = std::min(retryDelayMs_ * 2, kMaxRetryDelayMs);
  }
  else
  {
    LOG_DEBUG << "do not connect";
  }
}
```

### tcp client能不能多次connect

再回去看下能不能多次connect不同的地址

```cpp
void Connector::startInLoop()
{
  loop_->assertInLoopThread();
  assert(state_ == kDisconnected);
  if (connect_)
  {
    connect();
  }
  else
  {
    LOG_DEBUG << "do not connect";
  }
}
```

这里看到只要是disconnected的状态下，其实是可以多次connect，然后不断的创建新的connection的
