---
title: 虚假唤醒
date: 2026-03-16 14:55:00 +0800
categories: [Blogging, c++, os, thread, condition-variable, mutex]
tags: [writing]
---

### 引言


一条铁律：**“永远要在循环中检查等待条件。”** ，不然有概率碰到“虚假唤醒（Spurious Wakeup）”

---

### 一、 什么是虚假唤醒？
虚假唤醒是指线程在没有收到预期的 `signal` 或 `broadcast` 信号时，从 `wait` 状态返回了；或者虽然收到了信号，但由于竞争环境，苏醒后发现条件其实并不满足。

---

### 二、 追根溯源：Linux 系统内核视角

C++ 的 `std::condition_variable` 在 Linux 下本质上是对 `pthread_cond_wait` 的封装，而 `pthread` 又是对系统调用 `futex` 的封装。

#### 1. 为什么内核允许虚假唤醒？
在 POSIX 标准中，`pthread_cond_wait` 允许产生虚假唤醒，主要原因是**性能**。

*   **中断处理：** 在 Linux 中，当线程在 `futex` 系统调用中阻塞时，如果收到信号（Signal，如 `SIGUSR1`），系统调用会被中断。
*   **信号处理返回：** 内核要么自动重启系统调用，要么让系统调用返回 `EINTR`。为了简化底层实现并避免极其复杂的原子性保证，内核和线程库选择直接让用户态线程苏醒。
*   **多核一致性：** 在多处理器系统上，确保“只有一个线程被精确唤醒”且“状态完全同步”的代价极其高昂，可能需要在大范围内核代码中加锁。允许少量的虚假唤醒可以极大提高高并发下的吞吐量。

---

### 三、 C++ 标准库的封装


#### 1. 基础 `wait` 的实现
在 `condition_variable` 头文件中，最基础的 `wait` 实现如下：

```cpp
void wait(unique_lock<mutex>& __lock) noexcept {
    // 最终调用的是 POSIX 的 pthread_cond_wait
    __gthread_cond_wait(&_M_cond, __lock.mutex()->native_handle());
}
```
可以看到，这个接口是直接暴露原始接口的，它**不具备**抗虚假唤醒的能力。

#### 2. 带谓词（Predicate）的 `wait` 源码
这就是为什么 C++11 推荐使用带 Lambda 表达式的 `wait`。我们来看它的源码（摘自 libstdc++）：

```cpp
template<typename _Predicate>
void wait(unique_lock<mutex>& __lock, _Predicate __p) {
    // 重点：这是一个循环！
    while (!__p()) {
        wait(__lock); 
    }
}
```
**源码逻辑分析：**
1.  **检查条件：** 首先调用 `__p()`（你的 Lambda 表达式）。
2.  **进入等待：** 如果条件不满足，调用基础的 `wait(__lock)`。
3.  **苏醒检查：** 当线程苏醒（无论是真的信号还是虚假唤醒），它会回到 `while` 循环顶部再次执行 `!__p()`。
4.  **循环往复：** 如果是虚假唤醒，`__p()` 依然为 `false`，线程再次进入等待。只有当条件真正满足时，它才会退出循环。

---

### 四、 为什么 `if` 不够？—— 惊群效应与竞争

除了内核层面的虚假唤醒，逻辑层面也存在“类似虚假唤醒”的现象。

**场景描述：**
1.  线程 A 和 线程 B 都在等待一个资源。
2.  线程 C 产生了资源，调用 `notify_all()`。
3.  线程 A 和 线程 B 同时苏醒。
4.  **线程 A 抢先一步**夺取了互斥锁并消耗了资源。
5.  **线程 B 接着拿到锁**，如果此时 B 用的是 `if`，它会直接往下走，但资源已经被 A 取走了。

**结论：** 无论是内核导致的虚假唤醒，还是多线程竞争导致的条件失效，`while` 循环都是唯一的救星。

---

### 五、 最佳实践：如何编写健壮的代码

#### 错误范式（危险）：
```cpp
std::unique_lock<std::mutex> lk(mtx);
if (!ready) {
    cv.wait(lk); // 虚假唤醒后 ready 依然为 false，导致崩溃
}
// 处理资源...
```

#### 经典范式（推荐）：
```cpp
std::unique_lock<std::mutex> lk(mtx);
// 使用 Lambda 表达式，优雅且安全
cv.wait(lk, []{ return ready; }); 
// 处理资源...
```

---

### 总结
1.  **虚假唤醒是客观存在的：** 受限于操作系统实现和 POSIX 标准。
2.  **根源在内核：** 性能权衡和中断处理导致 `futex` 意外返回。
3.  **标准库的应对：** C++ `wait(lock, pred)` 内部通过 `while` 循环进行了封装。

### REF

+ [为什么条件锁会产生虚假唤醒现象](https://www.zhihu.com/question/271521213/answer/2015069599700382855)
