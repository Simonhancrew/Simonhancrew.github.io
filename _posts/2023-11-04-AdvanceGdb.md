---
title: Advance GDB
date: 2023-11-04 14:10:00 +0800
categories: [Blogging, debug]
tags: [writing]
---

more than breakpoint + frame + bt + as

主要关注一下他人的调试经验，以及一些高级调试技巧，比如使用ebpf进行性能分析，使用gdb进行内存分析等等。

另外xxdb调试，一个很关键的技巧，我个人觉得是要掌握data point和watch point的使用，这个可以参考[这里](https://www.cnblogs.com/liuhanxu/p/16123981.html), 另外ref的第5篇文章也有提到。

另外经常使用lldb，需要参考lldb到gdb的[map](https://lldb.llvm.org/use/map.html)

另外还有一个[apple官方的tutorial](https://developer.apple.com/library/archive/documentation/IDEs/Conceptual/gdb_to_lldb_transition_guide/document/lldb-command-examples.html)

### lldb稍微有作用点的方法

#### watchpoint

+ 内存断点 watchpoint set expression 地址

+ watchpoint set variable 变量名

1. 设置内存断点

```bash
(lldb) watchpoint set expression <address>
```

2. 内存访问断点

```bash
watchpoint set expression -w read -- 内存地址

(lldb) watchpoint set expression -w read -- 内存地址
```

3. 内存写入断点

```bash
watchpoint set expression -w write -- 内存地址

(lldb) watchpoint set expression -w read -- 内存地址
```

4. 条件断点

```bash
watchpoint modify -c 表达式
(lldb) watchpoint modify -c '*(int *)内存地址 == 20'
```

### coredump堆栈丢失和noexcept

[一剑破万法：noexcept与C++异常导致的coredump](https://zhuanlan.zhihu.com/p/609434714), 这个可以看下，复杂的回调函数里面，如果有异常，会导致coredump，这个时候bt不一定很清晰，虽然代码可以看出来，但是工程大了的花，一行行看代码其实有点耗时. 因为在函数没有声明noexcept的时候，异常会继续向调用函数抛出，直到遇到noexcept的函数，或者一直抛给main函数，然后触发coredump。

这个时候可以用noexcept，标注这个回调函数(lambda也可以用这个标注)。这样在coredump里，调用栈会稍微清晰一点。

原理：当你声明一个函数为 noexcept 时，你正在向编译器保证该函数不会抛出任何异常。如果在 noexcept 函数中发生了异常，程序会立即调用 std::terminate() 来进行异常的快速失败处理，通常这会导致程序的退出。


### Ref

1. [C++ 内存问题排查：创建 Zip 压缩包，解压后内容错乱](https://selfboot.cn/2023/10/19/c++_zip_memory_problem/)

2. [C++11 std::thread异常coredump导致调用堆栈丢失问题的跟踪和解决（std::teminate）](https://zhuanlan.zhihu.com/p/456536345)

3. [一剑破万法：noexcept与C++异常导致的coredump](https://zhuanlan.zhihu.com/p/609434714)

4. [复杂 C++ 项目堆栈保留以及 ebpf 性能分析](https://selfboot.cn/2023/10/17/c++_frame_pointer/)

5. [advanced-gdb](https://interrupt.memfault.com/blog/advanced-gdb)

6. [gdb小技巧汇总](https://www.cnblogs.com/liuhanxu/p/16026260.html)

7. [GDB观察点watchpoints的介绍和演示](https://www.cnblogs.com/liuhanxu/p/16123981.html)

8. [gdb调试多进程](https://www.cnblogs.com/liuhanxu/p/16158842.html)

9. [不同场景下使用gdb调试段错误的几种方法](https://www.cnblogs.com/liuhanxu/p/17446765.html)

10. [C++ STL有用？如何调试？](https://mp.weixin.qq.com/s/UWG-5WZLCavFNuwpfz6NaA)

11. [Apache arrow顶级项目调试](https://mp.weixin.qq.com/s/-XdastMGa3vVrN5nMiQkdw)

12. [干爆源码系列之Step by step lldb/gdb调试多线程](https://mp.weixin.qq.com/s/3-4KDcJgQ2cgqW_KALGstg)

13. [How to debug C and C++ programs with rr](https://developers.redhat.com/blog/2021/05/03/instant-replay-debugging-c-and-c-programs-with-rr#conclusion)

14. [gdb tricks](https://nullprogram.com/blog/2024/01/28/)
