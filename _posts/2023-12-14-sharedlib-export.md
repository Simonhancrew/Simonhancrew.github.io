---
layout: post
title:  "动态库符号导出"
category: compile
date:   2023-12-14
---

全平台动态库导出的方法，在[Ref1](#ref)里推荐的方法使用ld script。可以细看一下

## Linux平台动态库符号导出

linux下有几种符号导出的方式

### 使用attribute + visibility控制符号可见性

可以设置默认的visibility为hidden，然后针对要导出的接口的课件行设置为default，这样就可以控制符号的可见性了。寻常为了兼容性，一般会导出c的符号，如果用c++的话，要约定好版本

### 使用 static 关键字控制符号可见性

在C/C++语言中，static 关键字在不同场景下有不同意义，当使用 static 表示“该函数或变量仅在本文件可见”时，那么这个函数或变量就不会出现在动态符号表中，但只会删除其动态符号表项，而不会删除其实现体。static 关键字相当于是增强的 hidden（因为 static 声明的函数或变量编译时只对当前文件可见，而 hidden 声明的函数或变量只是在动态符号表中不存在，在编译期间对其他文件还是可见的）。在项目开发中，使用 static 关键字声明一个函数或变量“仅在本文件可见”是很好的习惯，但是不建议使用 static 关键字控制符号可见性：无法使用 static 关键字控制一个多文件可见的函数或变量的符号可见性。


### 使用 exclude libs 移除静态库中的符号

上述 visibility 方式、attribute 方式和 static 关键字，都是控制项目源码中符号的可见性，而无法控制依赖的静态库中的符号在最终 so 中是否存在。exclude libs 就是用来控制依赖的静态库中的符号是否可见，它是传递给链接器的参数，可以使依赖的静态库的符号在动态符号表中不存在。同样，也是只能删除符号表项，实现体仍然会存在于产生的 so 文件中。

### 使用version script

version script 是传递给链接器的参数，用来指定动态库导出哪些符号以及符号的版本。开启 version script 需要先编写一个文本文件，用来指定动态库导出哪些符号。


### 总结

看上去，version script 是明确地指定需要保留的符号，如果通过 visibility 结合 attribute 的方式控制每个符号是否导出，也能达到 version script 的效果，但是 version script 方式有一些额外的好处：

1. version script 方式可以控制编译进 so 的静态库的符号是否导出，visibility 和 attribute 方式都无法做到这一点。
2. visibility 结合 attribute 方式需要在源码中标明每个需要导出的符号，对于导出符号较多的项目来说是很繁杂的。version script 把需要导出的符号统一地放到了一起，能够直观方便地查看和修改，对导出符号较多的项目也非常友好。
3. version script 支持通配符，* 代表0个或者多个字符，? 代表单个字符。比如 my*; 就代表所有以 my 开头的符号。有了通配符的支持，配置 version script 会更加方便。
4. 还有非常特殊的一点，version script 方式可以删除 __bss_start 这样的一些符号（这是链接器默认加上的符号）。


## darwin下动态库符号导出方式


## windows下动态库符号导出方式


## Ref

1. [Android对so体积优化的探索与实践](https://tech.meituan.com/2022/06/02/meituans-technical-exploration-and-practice-of-android-so-volume-optimization.html)

2. [gcc doc](https://gcc.gnu.org/onlinedocs/gcc)

3. [LLVM Link Time Optimization: Design and Implementation](https://llvm.org/docs/LinkTimeOptimization.html)

4. [LD](https://sourceware.org/binutils/docs/ld/VERSION.html)

5. [LTO overview](https://gcc.gnu.org/onlinedocs/gccint/LTO-Overview.html)

6. [Executable and Linkable Format (ELF)](https://www.cs.cmu.edu/afs/cs/academic/class/15213-f00/docs/elf.pdf)
