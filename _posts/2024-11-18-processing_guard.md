---
title: processing guard
date: 2024-11-18 15:30:10 +0800
categories: [Blogging, callback]
tags: [writing]
---

主要是在callback之后，当前发起cb的类可能在cb之后被销毁，但是在cb之后还有一些操作需要做，这个时候要么shared_ptr来保障，要么自定义一个guard配合unique ptr的deleter做保障

```cpp
void ObjectA::Do() {}
  if (callback) {
    // NOTE: this callback may destroy the current object
    callback();
  }
  // NOTE: use after free
  member_.Write("");
  ...
}
```

一个简单的guard模块，配合自定一的release/destroy可以让uniqueptr看起来形如一个shared ptr

```cpp
class DelayedDestructor {
 public:
  class ProcessingGuard {
   public:
    ALWAYS_INLINE explicit ProcessingGuard(DelayedDestructor* delay_destructor)
        : destroy_interface_(delay_destructor),
          is_already_in_processing_(delay_destructor->is_in_processing_) {
      if (is_already_in_processing_) {
        return;
      }
      destroy_interface_->is_in_processing_ = true;
    }
    ALWAYS_INLINE ~ProcessingGuard() {
      if (is_already_in_processing_) {
        return;
      }
      destroy_interface_->is_in_processing_ = false;
      if (destroy_interface_->late_destroy_) {
        destroy_interface_->late_destroy_ = false;
        destroy_interface_->Destroy();
      }
    }
    ProcessingGuard(const ProcessingGuard&) = delete;
    ProcessingGuard& operator=(const ProcessingGuard&) = delete;
    ProcessingGuard(ProcessingGuard&&) = delete;
    ProcessingGuard& operator=(ProcessingGuard&&) = delete;

   private:
    bool is_already_in_processing_ = false;
    DelayedDestructor* destroy_interface_;
  };

  void Destroy() {
    if (is_in_processing_) {
      late_destroy_ = true;
    } else {
      delete this;
    }
  }

  virtual ~DelayedDestructor() = default;

 private:
  bool is_in_processing_ = false;
  bool late_destroy_ = false;
};

struct DestroyDeleter {
  ALWAYS_INLINE void operator()(DelayedDestructor* delay_destructor) const {
    if (delay_destructor) {
      delay_destructor->Destroy();
    }
  }
};
```

具体使用的时候只要继承DelayedDestructor，然后使用unique ptr即可

```cpp
A : public DelayedDestructor
using SafePtr = std::unique_ptr<A, DestroyDeleter>;
```

有些连接库我也看到了类似的处理逻辑，比如muduo里的channel，这里主要为了connection在处理的时候不析构，在establishconnection的时候就tie了一个connection的weak ptr

在后续处理的时候，保证在任意event处理中，不要析构connection。这里的作用也类似一个guard

```cpp
void Channel::tie(const std::shared_ptr<void>& obj)
{
  tie_ = obj;
  tied_ = true;
}

void Channel::handleEvent(Timestamp receiveTime)
{
  std::shared_ptr<void> guard;
  if (tied_)
  {
    guard = tie_.lock();
    if (guard)
    {
      handleEventWithGuard(receiveTime);
    }
  }
  else
  {
    handleEventWithGuard(receiveTime);
  }
}
```

