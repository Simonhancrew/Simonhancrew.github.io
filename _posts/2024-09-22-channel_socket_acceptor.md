---
title: channel socket acceptor设计
date: 2024-09-22 13:01:10 +0800
categories: [Blogging, channel, socket, acceptor, muduo]
tags: [writing]
---

## Channel

channel等于是一个fd需要多路复用情况下的封装，比如epoll在做多路服用的时候，需要epoll_ctl注册fd还有fd上感兴趣的事件，当事件发生的时候需要做相应的处理，这里channel实际上是封装了针对fd的事件处理，还有事件发生时的回调函数。

比如看下eventloop里wakeupfd那个channel的流程

```cpp
wakeupFd_(createEventfd()),
wakeupChannel_(new Channel(this, wakeupFd_)),
wakeupChannel_->setReadCallback(
    std::bind(&EventLoop::handleRead, this));
// we are always reading the wakeupfd
wakeupChannel_->enableReading();
```

具体的这两个函数`setReadCallback`和`enableReading`

```cpp
const int Channel::kNoneEvent = 0;
const int Channel::kReadEvent = POLLIN | POLLPRI;
const int Channel::kWriteEvent = POLLOUT;

void setReadCallback(ReadEventCallback cb)
{ readCallback_ = std::move(cb); }

void enableReading() { events_ |= kReadEvent; update(); }

void Channel::update()
{
  addedToLoop_ = true;
  loop_->updateChannel(this);
}

void EventLoop::updateChannel(Channel* channel)
{
  assert(channel->ownerLoop() == this);
  assertInLoopThread();
  poller_->updateChannel(channel);
}
```

这个时候再看channel的实现就比较清楚

## acceptor and socket

首先看下socket的封装，基本只有acceptor在用，acceptor也主要只有tcp server在用

基本都是os的socket api的封装

```cpp
///
/// Wrapper of socket file descriptor.
///
/// It closes the sockfd when desctructs.
/// It's thread safe, all operations are delagated to OS.
class Socket : noncopyable
{
 public:
 int fd() const { return sockfd_; }
  // return true if success.
  bool getTcpInfo(struct tcp_info*) const;
  bool getTcpInfoString(char* buf, int len) const;

  /// abort if address in use
  void bindAddress(const InetAddress& localaddr);
  /// abort if address in use
  void listen();

  /// On success, returns a non-negative integer that is
  /// a descriptor for the accepted socket, which has been
  /// set to non-blocking and close-on-exec. *peeraddr is assigned.
  /// On error, -1 is returned, and *peeraddr is untouched.
  int accept(InetAddress* peeraddr);

  void shutdownWrite();

  ///
  /// Enable/disable TCP_NODELAY (disable/enable Nagle's algorithm).
  ///
  void setTcpNoDelay(bool on);

  ///
  /// Enable/disable SO_REUSEADDR
  ///
  void setReuseAddr(bool on);

  ///
  /// Enable/disable SO_REUSEPORT
  ///
  void setReusePort(bool on);

  ///
  /// Enable/disable SO_KEEPALIVE
  ///
  void setKeepAlive(bool on);
}
```

然后看acceptor，里面还是对channel + listen fd的封装

+ `acceptSocket_`：这个是服务器监听套接字的文件描述符

+ `acceptChannel_`：这是个Channel类，把acceptSocket_及其感兴趣事件和事件对应的处理函数都封装进去。

+ `EventLoop *loop`：监听套接字的fd由哪个EventLoop负责循环监听以及处理相应事件，其实这个EventLoop就是main EventLoop。

+ `newConnectionCallback_`: TcpServer构造函数中将TcpServer::newConnection( )函数注册给了这个成员变量。这个TcpServer::newConnection函数的功能是依靠round robin把链接分发到subloop中去处理。

可以看下`handleread`的时候的处理，这里有一个要注意的点

```cpp

void Acceptor::handleRead()
{
  loop_->assertInLoopThread();
  InetAddress peerAddr;
  //FIXME loop until no more
  int connfd = acceptSocket_.accept(&peerAddr);
  if (connfd >= 0)
  {
    // string hostport = peerAddr.toIpPort();
    // LOG_TRACE << "Accepts of " << hostport;
    if (newConnectionCallback_)
    {
      newConnectionCallback_(connfd, peerAddr);
    }
    else
    {
      sockets::close(connfd);
    }
  }
  else
  {
    LOG_SYSERR << "in Acceptor::handleRead";
    // Read the section named "The special problem of
    // accept()ing when you can't" in libev's doc.
    // By Marc Lehmann, author of libev.
    if (errno == EMFILE)
    {
      ::close(idleFd_);
      idleFd_ = ::accept(acceptSocket_.fd(), NULL, NULL);
      ::close(idleFd_);
      idleFd_ = ::open("/dev/null", O_RDONLY | O_CLOEXEC);
    }
  }
}
```

最后的error处理有一个POSIX accept 函数的特殊行为：

+ 在某些错误情况下（如 2004 年后的 Linux 系统），accept 函数不会从待处理队列中移除连接。

找到的[原话](http://pod.tst.eu/http://cvs.schmorp.de/libev/ev.pod#The_special_problem_of_accept_ing_wh)是

```ascii
Many implementations of the POSIX accept function (for example, found in post-2004 Linux) have the peculiar behaviour of not removing a connection from the pending queue in all error cases.

For example, larger servers often run out of file descriptors (because of resource limits), causing accept to fail with ENFILE but not rejecting the connection, leading to libev signalling readiness on the next iteration again (the connection still exists after all), and typically causing the program to loop at 100% CPU usage.

Unfortunately, the set of errors that cause this issue differs between operating systems, there is usually little the app can do to remedy the situation, and no known thread-safe method of removing the connection to cope with overload is known (to me).

One of the easiest ways to handle this situation is to just ignore it - when the program encounters an overload, it will just loop until the situation is over. While this is a form of busy waiting, no OS offers an event-based way to handle this situation, so it's the best one can do.

A better way to handle the situation is to log any errors other than EAGAIN and EWOULDBLOCK, making sure not to flood the log with such messages, and continue as usual, which at least gives the user an idea of what could be wrong ("raise the ulimit!"). For extra points one could stop the ev_io watcher on the listening fd "for a while", which reduces CPU usage.

If your program is single-threaded, then you could also keep a dummy file descriptor for overload situations (e.g. by opening /dev/null), and when you run into ENFILE or EMFILE, close it, run accept, close that fd, and create a new dummy fd. This will gracefully refuse clients under typical overload conditions.

The last way to handle it is to simply log the error and exit, as is often done with malloc failures, but this results in an easy opportunity for a DoS attack.
```

这里单线程的处理方法

+ 保留一个用于过载情况的虚拟文件描述符（如打开 /dev/null）。
+ 遇到 ENFILE 或 EMFILE 错误时，关闭虚拟描述符，执行 accept，关闭新接受的描述符，然后重新创建虚拟描述符。
+ 这种方法可以在典型的过载条件下优雅地拒绝客户端连接。

所以里面那个idleFd_就是为了处理这个留着的
