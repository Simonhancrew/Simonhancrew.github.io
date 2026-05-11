---
title: lambda detroy carsh
date: 2026-05-11 16:31:49 +0800
categories: [Blogging, c++]
tags: [writing]
---


多年前写了一个测试组件，主要是模拟clock跟timer，最近发现他有崩溃

测试的代码如下

```cpp
testing::FakeClock clock;
int count = 0;
std::unique_ptr<commons::timer_base> timer;
timer.reset(clock.CreateTimer(
    [&count, &timer]() {
        timer.reset();  // delete self
        count++;
    },
    100, /*one_shot=*/true));
```

然后直接shedule 100ms，timer触发，之后就崩了

这里timer run起来的代码也很简单

```cpp
void FakeTimer::Run() {
  if (is_running_) {
    running_();
  }
}
```

这里没考虑到的问题是callback如果捕获了值，而且这个lambda可能直接析构掉自己，之后还去调用一下这个值

比较简单的解法就是直接把这个funtion拷贝一份，这样哪怕fake timer自己的running析构了，但是拷贝的那个在栈上调用的还好好的，之前capture的那个值也拷贝了一份

```cpp
void FakeTimer::Run() {
  if (is_running_) {
    // in callback, release timer first, function may destoy self, should copy running_ first
    auto cb = running_;
    cb();
  }
}
```

看起来是能处理大多数的情况的
