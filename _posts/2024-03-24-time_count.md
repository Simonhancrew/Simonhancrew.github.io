---
title: 计时问题
date: 2024-03-24 14:10:00 +0800
categories: [Blogging, cpp]
tags: [writing]
---

集中看下，c++代码下，有哪些可行的计时方法

## 低精度

### clock

这种主要使用c语言的`clock`函数，精度较低，但是可以用于简单的计时。返回的时间其实是个long_t, 单位是ms，所以需要除以`CLOCKS_PER_SEC`来得到秒数。

```cpp
#include <ctime>            // 需要包含头文件

clock_t start = clock();    // 开始时间
// do something
clock_t end = clock();      // 结束时间
std::cout << "Spent " << double(end - start) / CLOCKS_PER_SEC << " seconds." << std::endl;  // 输出时间（单位：s）
```

## 高精度

### c++ chrono

大多数的代码如果要用ms级别的计时，可以使用`std::chrono`库，这个库是c++11引入的，可以用于高精度的计时。示例代码可以随便写写

```cpp
inline uint64_t NowMs() {
  return std::chrono::duration_cast<std::chrono::milliseconds>(
             std::chrono::system_clock::now().time_since_epoch())
      .count();
}

inline uint64_t TickMs() {
  return std::chrono::duration_cast<std::chrono::milliseconds>(
             std::chrono::steady_clock::now().time_since_epoch())
      .count();
}
```

chrono主要掌握三个概念就可以：

1. 持续时间(duration): `std::chrono::duration<Rep, Period>`, `Rep`是一个整数类型，`Period`是一个`std::ratio`类型，表示`Rep`的单位。比如`std::chrono::milliseconds`表示`Rep`是`long long`，`Period`是`std::milli`，表示ms。
2. 时间点(time_point): `std::chrono::time_point<Clock, Duration>`, `Clock`是一个时钟类型，`Duration`是一个持续时间类型。比如`std::chrono::system_clock::time_point`表示系统时钟的时间点。
3. 时钟(clock): `std::chrono::system_clock`, `std::chrono::steady_clock`, `std::chrono::high_resolution_clock`等等。

一般来说，`std::chrono::system_clock`是系统时钟，`std::chrono::steady_clock`是稳定时钟，`std::chrono::high_resolution_clock`是高精度时钟，可能是`std::chrono::system_clock`或者`std::chrono::steady_clock`的别名。

## 一些时间上的概念

### wall clock time

Wall Clock Time 是一段代码在某个线程上实际执行的时间，但由于cpu是分时间片给线程的，同时一段代码可能由于IO，还是类似调用wait等线程调用方法，阻塞不执行了，此时cpu会分配给其它线程，但这段代码其实并没有执行完，可能等某个条件触发后，轮到这段代码在的线程分到cpu后，继续执行，而执行这段代码cpu真正的用时是Thread Time。

### cpu time

指的是计算机处理器在执行一个特定程序时花费的时间，也就是程序在处理器上实际运行的时间。

### clock time

指的是程序从开始执行到结束所花费的时间，包括了等待资源、I/O 操作等等与 CPU 时间无关的时间.

## REF

1. [Measure time in Linux - time vs clock vs getrusage vs clock_gettime vs gettimeofday vs timespec_get](https://stackoverflow.com/questions/12392278/measure-time-in-linux-time-vs-clock-vs-getrusage-vs-clock-gettime-vs-gettimeof)
2. [如何从Wall/CPU time理解多线程程序的并行效率](https://zhuanlan.zhihu.com/p/39891521)
3. [C++20: Basic Chrono Terminology with Time Duration and Time Point](https://www.modernescpp.com/index.php/c20-basic-chrono-terminology-with-time-duration-and-time-point/)