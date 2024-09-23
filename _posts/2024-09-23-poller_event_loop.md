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
