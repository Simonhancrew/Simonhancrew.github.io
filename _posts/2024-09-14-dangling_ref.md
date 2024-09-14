---
title: dangling reference
date: 2024-09-14 17:01:10 +0800
categories: [Blogging, debug]
tags: [writing]
---

比较复杂的回调里经常出现析构父对象之后还在继续使用父对象的行为

这种就属于悬垂引用问题，针对这种问题开asan都能快速崩掉

这种问题可能导致问题出现乱蹦的堆栈，或者你开了asan之后，一直崩在malloc里，这个时候要做的其实就是**缩小调查的范围**

1. 如果堆栈直接能看出问题，那当然最好了，崩的点就直接是问题的点
2. 如果堆栈不清晰，就要配合复现的日志，最好是多个，对比定位到相似的日志，缩小到出问题的模块
3. 定位到模块之后，这种悬垂引用使用问题，一般是析构 + 重用，一般会有在callback函数里，所以先找callback
4. 如果3依然没有一眼看出来，就要用下面的试试了，通过watchpoint + 地址打印来调试悬垂引用

具体：

1. **启动调试会话**
   - 编译程序时加入调试信息（使用 `-g` 标志）。
   - 在 LLDB 或 GDB 中加载你的程序。

2. **设置初始断点**
   - 在 `main` 函数或程序的入口点设置断点。
   - 运行程序直到这个断点。

3. **设置 watchpoint**
   - 对于 LLDB:
     ```
     (lldb) watchpoint set variable foo
     ```
   - 对于 GDB:
     ```
     (gdb) watch foo
     ```
   这会在 `foo` 变量的地址上设置一个 watchpoint。

4. **运行程序并观察 watchpoint**
   - 继续运行程序。每次 `foo` 的值改变时，调试器都会停下来。

5. **记录地址变化**
   - 当 watchpoint 触发时，记录 `foo` 的旧值和新值。
   - 在 LLDB 中：
     ```
     (lldb) p foo
     (lldb) p (void*)foo.get()  // 获取原始指针地址
     ```
   - 在 GDB 中：
     ```
     (gdb) p foo
     (gdb) p foo.get()
     ```

6. **设置断点在回调函数中**
   - 在 `Foo::Bar` 方法中设置断点，特别是在悬垂掉用这行。

7. **检查回调中使用的对象地址**
   - 当程序在回调函数中停下时，检查正在使用的 `this` 指针：
     - LLDB: `p (void*)this`
     - GDB: `p this`
   - 比较这个地址与之前记录的 `foo` 地址。

8. **使用条件 watchpoint**
   - 如果你知道问题发生在特定情况下，可以设置条件 watchpoint：
     - LLDB: `watchpoint modify -c '(void*)foo.get() == 0x12345678'`
     - GDB: `watch foo if foo.get() == 0x12345678`
   - 这里的 `0x12345678` 是你观察到的可能导致问题的特定地址。

9. **分析调用栈**
   - 每次 watchpoint 触发或在回调中停止时，检查调用栈：
     - LLDB: `bt`
     - GDB: `backtrace`
   - 这有助于理解 `foo` 被重置和访问的上下文。

10. **使用脚本自动化**
    - 在 LLDB 中，你可以使用 Python 脚本来自动化这个过程：
      ```python
      def on_watchpoint(frame, wp, dict):
          old_addr = frame.EvaluateExpression("(void*)foo.get()").unsigned
          frame.EvaluateExpression("foo.reset(new Foo)")
          new_addr = frame.EvaluateExpression("(void*)foo.get()").unsigned
          print(f"foo reset: {old_addr:x} -> {new_addr:x}")
          return False

      watchpoint = target.WatchAddress(foo_address, 8, False, True, on_watchpoint)
      ```

根本点就是确定每一个析构掉的地址，在后面都不能重新用了

目前我看下来，大多数都要缩小范围看下去，可能要试试有啥方法能直接看堆栈接能解决掉了，或者还有别的别的能用的调试信息，不仅仅是dump，能在第一步就看出来问题在哪里了
