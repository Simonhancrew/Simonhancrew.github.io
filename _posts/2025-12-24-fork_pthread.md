---
title: 定位fork之后存在的死锁问题
date: 2025-12-24 16:30:00 +0800
categories: [Blogging, posix, pthread, fork, deadlock]
tags: [writing]
---

### 起因

在apple平台下，我们新加了一个Network.framework，用于探测一些系统连接的状态，这里我用使用了nw_path_monitor_create来创建一个path monitor，并在一个单独的线程中运行。

随后在ut中会大概率报死锁问题

结果如下:

```
*** multi-threaded process forked ***
BUG IN CLIENT OF LIBPLATFORM: os_unfair_lock is corrupt, or owner thread exited without unlocking
Abort Cause 3846
crashed on child side of fork pre-exec
```

最后几行的stacktrace如下:

```
thread 0 crashed：
0  libsystem_platform.dylib       0x00000001a4f4e2b0 _os_unfair_lock_lock_abort + 52
1  libsystem_platform.dylib       0x00000001a4f4e1d8 os_unfair_lock_lock + 24
2  libnetwork.dylib               0x00000001a5b1e    4 nw_path_shared_netcp_fd(NSObject<OS_nw_context>*) + 112
....
8 NetWork.framework           0x00000001a5b2c7d8 nw_path_release_globals + 176
9 NetWork.framework           0x00000001a5b2c9b8 nw_settings_child_has_forked + 88
10 libsystem_pthread.dylib        0x00000001a4f9f1c8 _pthread_at_fork_child_handler + 104
11 libsystem_c.dylib              0x00000001a4e5d fork + 64
```

可以清楚的看到的是fork之后，pthread注册了at_fork的handler，然后遇到了死锁问题

这里查到的官方解答是

```
There’s a fundamental disconnect between BSD and Mach on this topic, and Apple’s frameworks rely on Mach a lot
```

解决方案是用posix_spawn来替代fork+exec, 随后我们在ut里用posix_spawn替代fork+exec就没有再出现死锁问题

### fork问题

在传统的 Unix 系统中：

```c
pid_t pid = fork();
if (pid == 0) {
    // child
    exec(...);
}
```

- `fork()` 会复制父进程的整个地址空间（包括所有线程的状态）
- 但**子进程只继承调用 fork() 的那个线程**，其他线程“凭空消失”
- 如果其他线程当时正持有某个全局锁（比如 malloc 的内部锁），这个锁在子进程中就**永远无法释放** → 死锁！

#### 👇 示例重现：

```
parent                          child
======                          =====
thread A        thread B      
--------        --------      
malloc          fork() → child starts
> lock global_malloc_lock       |
                                v
                            malloc()
                            > tries to lock global_malloc_lock
                            ⛔ DEADLOCK —— 因为 thread A 不在 child 中，没人能解锁！
```

👉 这就是经典的 “**fork 在多线程程序中的不安全行为**”。

---

### 🔧 POSIX 的补救方案：`pthread_atfork`

为了解决上述问题，POSIX 引入了：

```c
int pthread_atfork(void (*prepare)(void),
                   void (*parent)(void),
                   void (*child)(void));
```

作用是：

- `prepare()`：在 fork 前调用（通常用于加锁，防止状态被破坏）
- `parent()`：fork 后在父进程中调用（通常用于解锁）
- `child()`：fork 后在子进程中调用（清理或重置状态，比如释放锁、重置随机数种子等）

⚠️ **但是！**

1. 很多库开发者**不知道要注册这个函数**
2. 即使注册了，也很容易写错（因为 child handler 只能调用 async-signal-safe 函数，限制极严）
3. 比如你在 child handler 里调用了 `printf()` 或 `malloc()` —— ❗这是未定义行为！

> 📌 man page 明确说：
> ```
> Important: only async-signal-safe functions are allowed on the child side of fork.
> ```

也就是说，在 fork 后的子进程中，你只能使用像 `write()`, `_exit()`, `signal()` 这种极少数“信号安全”的函数 —— 不能分配内存、不能打印、不能调用大多数标准库函数！

这极大限制了可用性 → 实际上很难正确实现。

因此，上述的思索问题，实际上很大程度上源于这个设计缺陷，在apple的平台下，这个atfork的handler实现估计是有什么缺陷，会导致死锁问题，就像他们官方说的，这个handler很难实现✅。

---

### ✅ 更好的替代方案：`posix_spawn`

于是出现了更现代、更安全的方式 —— **`posix_spawn`**

它的设计哲学完全不同：

> **不经过“中间态”（forked-but-not-exec’d）—— 直接从内核创建新进程并加载程序，跳过用户态复制阶段。**

### 💡 关键优势：

- ❌ **父进程从不 fork** —— 所以不存在“复制多线程状态”的问题
- ✅ **子进程直接从 exec 点开始运行** —— 没有中间状态，没有锁残留
- 🚫 **不会触发 pthread_atfork 逻辑** —— 因为根本没走 fork 路径
- ⚡ **原子性创建进程** —— 更高效、更安全

这就是文中说的：

> “The child process just spontaneously, and atomically, pops into existence.”

→ 子进程像是“瞬间+原子地”诞生了，完全避开了传统 fork 的坑。

---

### 🍎 NSTask（macOS/iOS）

Objective-C / Swift 中的 `NSTask`（现名 `Process`），我估计底层也是调用了 `posix_spawn`，因为它同样避免了 fork 的问题。

### 总结

核心来讲，这其实是Mach内核设计与传统Unix模型的冲突导致的问题。

BSD 是传统 Unix，基于 fork/exec 模型
Mach 是微内核架构（macOS/iOS 底层是 XNU = Mach + BSD），倾向于轻量级任务和消息传递

pthread + fork的这种组合在Mach下其实是非常不推荐的方式，尤其是当多线程遇到Mach api的时候
