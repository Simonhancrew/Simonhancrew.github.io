---
title: loop和poller设计
date: 2024-09-23 13:01:10 +0800
categories: [Blogging, event loop, poller, muduo]
tags: [writing]
---

### poller

poller是一个监听组件，接口非常的简单，配合eventloop使用。 在multi reactor的结构中，有多少个reactor就有多少个poller

```cpp
class Poller : noncopyable
{
 public:
  typedef std::vector<Channel*> ChannelList;

  Poller(EventLoop* loop);
  virtual ~Poller();

  /// Polls the I/O events.
  /// Must be called in the loop thread.
  virtual Timestamp poll(int timeoutMs, ChannelList* activeChannels) = 0;

  /// Changes the interested I/O events.
  /// Must be called in the loop thread.
  virtual void updateChannel(Channel* channel) = 0;

  /// Remove the channel, when it destructs.
  /// Must be called in the loop thread.
  virtual void removeChannel(Channel* channel) = 0;

  virtual bool hasChannel(Channel* channel) const;

  static Poller* newDefaultPoller(EventLoop* loop);

  void assertInLoopThread() const
  {
    ownerLoop_->assertInLoopThread();
  }

 protected:
  typedef std::map<int, Channel*> ChannelMap;
  ChannelMap channels_;

 private:
  EventLoop* ownerLoop_;
};
```

poller在loop中负责监听文件描述符事件是否触发以及返回发生事件的文件描述符以及具体事件

比如我们看epoller的loop调用

```cpp
Timestamp EPollPoller::poll(int timeoutMs, ChannelList* activeChannels)
{
  LOG_TRACE << "fd total count " << channels_.size();
  int numEvents = ::epoll_wait(epollfd_,
                               &*events_.begin(),
                               static_cast<int>(events_.size()),
                               timeoutMs);
  int savedErrno = errno;
  Timestamp now(Timestamp::now());
  if (numEvents > 0)
  {
    LOG_TRACE << numEvents << " events happened";
    fillActiveChannels(numEvents, activeChannels);
    if (implicit_cast<size_t>(numEvents) == events_.size())
    {
      events_.resize(events_.size()*2);
    }
  }
  else if (numEvents == 0)
  {
    LOG_TRACE << "nothing happened";
  }
  else
  {
    // error happens, log uncommon ones
    if (savedErrno != EINTR)
    {
      errno = savedErrno;
      LOG_SYSERR << "EPollPoller::poll()";
    }
  }
  return now;
}

void EPollPoller::fillActiveChannels(int numEvents,
                                     ChannelList* activeChannels) const
{
  assert(implicit_cast<size_t>(numEvents) <= events_.size());
  for (int i = 0; i < numEvents; ++i)
  {
    Channel* channel = static_cast<Channel*>(events_[i].data.ptr);
#ifndef NDEBUG
    int fd = channel->fd();
    ChannelMap::const_iterator it = channels_.find(fd);
    assert(it != channels_.end());
    assert(it->second == channel);
#endif
    channel->set_revents(events_[i].events);
    activeChannels->push_back(channel);
  }
}
```

每次都有一个空的active channels进来，从epollwait里拿到感兴趣的events，fill这个activeChannels，后面在上层对这些channel做操作。

这里需要设置每个channel的revents，来确定到底触发的是什么事件

针对每个channel，需要在之前主动注册到poller里，这里有一个updateChannel的方法，每个poller里有一个fd做key，channel做value的map，通过epoll_ctl来注册感兴趣的事件（这个event已经在channel里记录过了）

```cpp
void EPollPoller::updateChannel(Channel* channel)
{
  Poller::assertInLoopThread();
  const int index = channel->index();
  LOG_TRACE << "fd = " << channel->fd()
    << " events = " << channel->events() << " index = " << index;
  if (index == kNew || index == kDeleted)
  {
    // a new one, add with EPOLL_CTL_ADD
    int fd = channel->fd();
    if (index == kNew)
    {
      assert(channels_.find(fd) == channels_.end());
      channels_[fd] = channel;
    }
    else // index == kDeleted
    {
      assert(channels_.find(fd) != channels_.end());
      assert(channels_[fd] == channel);
    }

    channel->set_index(kAdded);
    update(EPOLL_CTL_ADD, channel);
  }
  else
  {
    // update existing one with EPOLL_CTL_MOD/DEL
    int fd = channel->fd();
    (void)fd;
    assert(channels_.find(fd) != channels_.end());
    assert(channels_[fd] == channel);
    assert(index == kAdded);
    if (channel->isNoneEvent())
    {
      update(EPOLL_CTL_DEL, channel);
      channel->set_index(kDeleted);
    }
    else
    {
      update(EPOLL_CTL_MOD, channel);
    }
  }
}

void EPollPoller::update(int operation, Channel* channel)
{
  struct epoll_event event;
  memZero(&event, sizeof event);
  event.events = channel->events();
  event.data.ptr = channel;
  int fd = channel->fd();
  LOG_TRACE << "epoll_ctl op = " << operationToString(operation)
    << " fd = " << fd << " event = { " << channel->eventsToString() << " }";
  if (::epoll_ctl(epollfd_, operation, fd, &event) < 0)
  {
    if (operation == EPOLL_CTL_DEL)
    {
      LOG_SYSERR << "epoll_ctl op =" << operationToString(operation) << " fd =" << fd;
    }
    else
    {
      LOG_SYSFATAL << "epoll_ctl op =" << operationToString(operation) << " fd =" << fd;
    }
  }
}
```

当然这里poller不持有channel，生命周期还是在上层管理

### eventloop

eventloop是最核心的事件循环体，它持有poller，负责调用poller的poll方法，然后处理返回的active channels。

这里主要有两大类方法

  1. loop, 占有线程并且做事件循环
  2. 需要主动调用的方法，注册一下方法到poller里，比如timer跟sync/async做函数调用

```cpp
///
/// Loops forever.
///
/// Must be called in the same thread as creation of the object.
///
void loop();

// timers

///
/// Runs callback at 'time'.
/// Safe to call from other threads.
///
TimerId runAt(Timestamp time, TimerCallback cb);
///
/// Runs callback after @c delay seconds.
/// Safe to call from other threads.
///
TimerId runAfter(double delay, TimerCallback cb);
///
/// Runs callback every @c interval seconds.
/// Safe to call from other threads.
///
TimerId runEvery(double interval, TimerCallback cb);

/// Runs callback immediately in the loop thread.
/// It wakes up the loop, and run the cb.
/// If in the same loop thread, cb is run within the function.
/// Safe to call from other threads.
void runInLoop(Functor cb);
/// Queues callback in the loop thread.
/// Runs after finish pooling.
/// Safe to call from other threads.
void queueInLoop(Functor cb);
```

首先看eventloop的构造

```cpp
EventLoop::EventLoop()
  : looping_(false),
    quit_(false),
    eventHandling_(false),
    callingPendingFunctors_(false),
    iteration_(0),
    threadId_(CurrentThread::tid()),
    poller_(Poller::newDefaultPoller(this)),
    timerQueue_(new TimerQueue(this)),
    wakeupFd_(createEventfd()),
    wakeupChannel_(new Channel(this, wakeupFd_)),
    currentActiveChannel_(NULL)
{
  LOG_DEBUG << "EventLoop created " << this << " in thread " << threadId_;
  if (t_loopInThisThread)
  {
    LOG_FATAL << "Another EventLoop " << t_loopInThisThread
              << " exists in this thread " << threadId_;
  }
  else
  {
    t_loopInThisThread = this;
  }
  wakeupChannel_->setReadCallback(
      std::bind(&EventLoop::handleRead, this));
  // we are always reading the wakeupfd
  wakeupChannel_->enableReading();
}
```

其中，CurrentThread::tid()会缓存当前的线程id。

具体的

```cpp
// internal
extern __thread int t_cachedTid;
extern __thread char t_tidString[32];
extern __thread int t_tidStringLength;
extern __thread const char* t_threadName;
void cacheTid();

inline int tid()
{
  if (__builtin_expect(t_cachedTid == 0, 0))
  {
    cacheTid();
  }
  return t_cachedTid;
}

void CurrentThread::cacheTid()
{
  if (t_cachedTid == 0)
  {
    t_cachedTid = detail::gettid();
    t_tidStringLength = snprintf(t_tidString, sizeof t_tidString, "%5d ", t_cachedTid);
  }
}

pid_t gettid()
{
  return static_cast<pid_t>(::syscall(SYS_gettid));
}
```

默认创造出来的poller是epoller

```cpp
Poller* Poller::newDefaultPoller(EventLoop* loop)
{
  if (::getenv("MUDUO_USE_POLL"))
  {
    return new PollPoller(loop);
  }
  else
  {
    return new EPollPoller(loop);
  }
}
```

其中`wakeupChannel_`是用来打断epoll loop的（是一个读的eventfd），具体看一下loop的实现

```cpp
void EventLoop::loop()
{
  assert(!looping_);
  assertInLoopThread();
  looping_ = true;
  quit_ = false;  // FIXME: what if someone calls quit() before loop() ?
  LOG_TRACE << "EventLoop " << this << " start looping";

  while (!quit_)
  {
    activeChannels_.clear();
    pollReturnTime_ = poller_->poll(kPollTimeMs, &activeChannels_);
    ++iteration_;
    if (Logger::logLevel() <= Logger::TRACE)
    {
      printActiveChannels();
    }
    // TODO sort channel by priority
    eventHandling_ = true;
    for (Channel* channel : activeChannels_)
    {
      currentActiveChannel_ = channel;
      currentActiveChannel_->handleEvent(pollReturnTime_);
    }
    currentActiveChannel_ = NULL;
    eventHandling_ = false;
    doPendingFunctors();
  }

  LOG_TRACE << "EventLoop " << this << " stop looping";
  looping_ = false;
}

void EventLoop::doPendingFunctors()
{
  std::vector<Functor> functors;
  callingPendingFunctors_ = true;

  {
  MutexLockGuard lock(mutex_);
  functors.swap(pendingFunctors_);
  }

  for (const Functor& functor : functors)
  {
    functor();
  }
  callingPendingFunctors_ = false;
}
```

判断是不是同线程call进来的, 这个`threadId_`是在构造的时候绑定到当前loop的

```cpp
bool isInLoopThread() const { return threadId_ == CurrentThread::tid(); }
```

加入此时有一个queueInLoop的函数调用，会被放到pendingFunctors里，然后在loop里面调用

```cpp
void EventLoop::queueInLoop(Functor cb)
{
  {
  MutexLockGuard lock(mutex_);
  pendingFunctors_.push_back(std::move(cb));
  }

  if (!isInLoopThread() || callingPendingFunctors_)
  {
    wakeup();
  }
}

void EventLoop::wakeup()
{
  uint64_t one = 1;
  ssize_t n = sockets::write(wakeupFd_, &one, sizeof one);
  if (n != sizeof one)
  {
    LOG_ERROR << "EventLoop::wakeup() writes " << n << " bytes instead of 8";
  }
}
```

这里的调用会先推到pendingFunctors_里，不在同线程或者当前已经在处理pendingfunctor的时候需要在唤醒一次loop

另外runInLoop和queInLoop的区别就是如果当前是同线程runInLoop，会立即执行，否则的话就是放到pendingFunctors里再去执行
