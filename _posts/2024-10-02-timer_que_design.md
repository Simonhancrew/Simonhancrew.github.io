---
title: timer queue design
date: 2024-10-02 10:01:10 +0800
categories: [Blogging, muduo, timer]
tags: [writing]
---

timer的设计在linux下看起来是通用的，大部分都是用的timerfd() + 一个数据结构维护timers的到期时间，其中timerfd是最小的超时时间(expiration)

以muduo为例看一下, 大多数的io库在linux上的处理都是类似的

接口

```cpp
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
///
/// Cancels the timer.
/// Safe to call from other threads.
///
void cancel(TimerId timerId);

// IMPL
TimerId EventLoop::runAt(Timestamp time, TimerCallback cb)
{
  return timerQueue_->addTimer(std::move(cb), time, 0.0);
}

TimerId EventLoop::runAfter(double delay, TimerCallback cb)
{
  Timestamp time(addTime(Timestamp::now(), delay));
  return runAt(time, std::move(cb));
}

TimerId EventLoop::runEvery(double interval, TimerCallback cb)
{
  Timestamp time(addTime(Timestamp::now(), interval));
  return timerQueue_->addTimer(std::move(cb), time, interval);
}

void EventLoop::cancel(TimerId timerId)
{
  return timerQueue_->cancel(timerId);
}
```

首先看下TimerId和里面的Timer到底是怎么封装的

```cpp
class TimerId : public muduo::copyable
{
 public:
  TimerId()
    : timer_(NULL),
      sequence_(0)
  {
  }

  TimerId(Timer* timer, int64_t seq)
    : timer_(timer),
      sequence_(seq)
  {
  }

  // default copy-ctor, dtor and assignment are okay

  friend class TimerQueue;

 private:
  Timer* timer_;
  int64_t sequence_;
};

typedef std::function<void()> TimerCallback;
class Timer : noncopyable
{
 public:
  Timer(TimerCallback cb, Timestamp when, double interval)
    : callback_(std::move(cb)),
      expiration_(when),
      interval_(interval),
      repeat_(interval > 0.0),
      sequence_(s_numCreated_.incrementAndGet())
  { }

  void run() const
  {
    callback_();
  }

  Timestamp expiration() const  { return expiration_; }
  bool repeat() const { return repeat_; }
  int64_t sequence() const { return sequence_; }

  void restart(Timestamp now);

  static int64_t numCreated() { return s_numCreated_.get(); }

 private:
  const TimerCallback callback_;
  Timestamp expiration_;
  const double interval_;
  const bool repeat_;
  const int64_t sequence_;

  static AtomicInt64 s_numCreated_;
};
```

可以看到的是timerid只是简单的包了一下timer和seq id

真实带有上下文信息的其实是timer，里面有回调和和当前已经create的timer数量，这里的s_numCreated_是一个static的原子变量，用来记录已经创建的timer数量，进程唯一

然后要看下timerqueue的实现

```cpp
///
/// A best efforts timer queue.
/// No guarantee that the callback will be on time.
///
class TimerQueue : noncopyable
{
 public:
  explicit TimerQueue(EventLoop* loop);
  ~TimerQueue();

  ///
  /// Schedules the callback to be run at given time,
  /// repeats if @c interval > 0.0.
  ///
  /// Must be thread safe. Usually be called from other threads.
  TimerId addTimer(TimerCallback cb,
                   Timestamp when,
                   double interval);

  void cancel(TimerId timerId);

 private:

  // FIXME: use unique_ptr<Timer> instead of raw pointers.
  // This requires heterogeneous comparison lookup (N3465) from C++14
  // so that we can find an T* in a set<unique_ptr<T>>.
  typedef std::pair<Timestamp, Timer*> Entry;
  typedef std::set<Entry> TimerList;
  typedef std::pair<Timer*, int64_t> ActiveTimer;
  typedef std::set<ActiveTimer> ActiveTimerSet;

  void addTimerInLoop(Timer* timer);
  void cancelInLoop(TimerId timerId);
  // called when timerfd alarms
  void handleRead();
  // move out all expired timers
  std::vector<Entry> getExpired(Timestamp now);
  void reset(const std::vector<Entry>& expired, Timestamp now);

  bool insert(Timer* timer);

  EventLoop* loop_;
  const int timerfd_;
  Channel timerfdChannel_;
  // Timer list sorted by expiration
  TimerList timers_;

  // for cancel()
  ActiveTimerSet activeTimers_;
  bool callingExpiredTimers_; /* atomic */
  ActiveTimerSet cancelingTimers_;
};
```

在构造的时候讲timerfd的channel准备好

```cpp
TimerQueue::TimerQueue(EventLoop* loop)
  : loop_(loop),
    timerfd_(createTimerfd()),
    timerfdChannel_(loop, timerfd_),
    timers_(),
    callingExpiredTimers_(false)
{
  timerfdChannel_.setReadCallback(
      std::bind(&TimerQueue::handleRead, this));
  // we are always reading the timerfd, we disarm it with timerfd_settime.
  timerfdChannel_.enableReading();
}
```

实际上每次addtimer的操作，在insert一个timer之后，判断最早到期时间是否改变，如果改变则重置timerfd的到期时间

```cpp
TimerId TimerQueue::addTimer(TimerCallback cb,
                             Timestamp when,
                             double interval)
{
  Timer* timer = new Timer(std::move(cb), when, interval);
  loop_->runInLoop(
      std::bind(&TimerQueue::addTimerInLoop, this, timer));
  return TimerId(timer, timer->sequence());
}

void TimerQueue::addTimerInLoop(Timer* timer)
{
  loop_->assertInLoopThread();
  bool earliestChanged = insert(timer);

  if (earliestChanged)
  {
    resetTimerfd(timerfd_, timer->expiration());
  }
}


bool TimerQueue::insert(Timer* timer)
{
  loop_->assertInLoopThread();
  assert(timers_.size() == activeTimers_.size());
  bool earliestChanged = false;
  Timestamp when = timer->expiration();
  TimerList::iterator it = timers_.begin();
  if (it == timers_.end() || when < it->first)
  {
    earliestChanged = true;
  }
  {
    std::pair<TimerList::iterator, bool> result
      = timers_.insert(Entry(when, timer));
    assert(result.second); (void)result;
  }
  {
    std::pair<ActiveTimerSet::iterator, bool> result
      = activeTimers_.insert(ActiveTimer(timer, timer->sequence()));
    assert(result.second); (void)result;
  }

  assert(timers_.size() == activeTimers_.size());
  return earliestChanged;
}
```

那么随后在唤醒的时候handleRead()中，会将到期的timer取出来，然后执行, 随后重置下一次timerfd的到期时间

这里handleread使用二分找到到期的timer，然后将其从timers_和activeTimers_中删除

```cpp
void TimerQueue::handleRead()
{
  loop_->assertInLoopThread();
  Timestamp now(Timestamp::now());
  readTimerfd(timerfd_, now);

  std::vector<Entry> expired = getExpired(now);

  callingExpiredTimers_ = true;
  cancelingTimers_.clear();
  // safe to callback outside critical section
  for (const Entry& it : expired)
  {
    it.second->run();
  }
  callingExpiredTimers_ = false;

  reset(expired, now);
}

std::vector<TimerQueue::Entry> TimerQueue::getExpired(Timestamp now)
{
  assert(timers_.size() == activeTimers_.size());
  std::vector<Entry> expired;
  Entry sentry(now, reinterpret_cast<Timer*>(UINTPTR_MAX));
  TimerList::iterator end = timers_.lower_bound(sentry);
  assert(end == timers_.end() || now < end->first);
  std::copy(timers_.begin(), end, back_inserter(expired));
  timers_.erase(timers_.begin(), end);

  for (const Entry& it : expired)
  {
    ActiveTimer timer(it.second, it.second->sequence());
    size_t n = activeTimers_.erase(timer);
    assert(n == 1); (void)n;
  }

  assert(timers_.size() == activeTimers_.size());
  return expired;
}
```

所以回头去看一下timerid是用什么数据结构存储的，c++的内置set，实际就是一个红黑树

```cpp
// FIXME: use unique_ptr<Timer> instead of raw pointers.
// This requires heterogeneous comparison lookup (N3465) from C++14
// so that we can find an T* in a set<unique_ptr<T>>.
typedef std::pair<Timestamp, Timer*> Entry;
typedef std::pair<Timer*, int64_t> ActiveTimer;
typedef std::set<ActiveTimer> ActiveTimerSet;
typedef std::set<Entry> TimerList;
// Timer list sorted by expiration
TimerList timers_;
// for cancel()
ActiveTimerSet activeTimers_;
bool callingExpiredTimers_; /* atomic */
ActiveTimerSet cancelingTimers_;
```

这里其实有好几个数据结构维护timers，其中timers_是为了维护一个顺序的到期时间timers，然后activeTimers_是为了维护一个timer的id和timer的映射，这样在cancel的时候可以直接找到timer
