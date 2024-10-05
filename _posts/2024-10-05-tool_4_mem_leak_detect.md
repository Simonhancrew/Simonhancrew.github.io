---
title: mem leakage detect tool
date: 2024-10-05 13:01:10 +0800
categories: [Blogging, tools, memory leak, memory]
tags: [writing]
---

主要看下mem leak检测的tool，一共有4个，分别是：

1. valgrind
2. AddressSanitizer
3. gperftools
4. bcc

## 1. valgrind

+ [valgrind](https://valgrind.org/) 

是一个非常老牌的工具组，其中的 memcheck 可以用来检测内存泄漏。但使用上不是很方便，是侵入式的，而且会对进程的性能有较大影响

## 2. Asan

+ [AddressSanitizer](https://github.com/google/sanitizers/wiki/AddressSanitizer)

是 gcc 4.8 开始内置支持的一个内存错误检测工具，仍然是侵入式的，对进程影响稍小（官方自夸）

其实我做LC的时候，c++是开了Asan的，c++的性能比GO还差

```ascii
AddressSanitizer 也被开启来检测 out-of-bounds 和 use-after-free 错误。
```

## 3. gperftools

+ [gperftools](https://github.com/gperftools/gperftools)

是一组高性能的支持多线程`malloc()`实现的集合，并附带了一些性能分析工具。

同样是侵入式的，可以把它链接到项目中，复用`tcmalloc`的能力去排查内存泄漏的地方，对进程影响稍小.

依赖`libunwind`, 自己装一下

在编译完成后会生成`tcmalloc`动态库和`pprof`脚本工具（用来解析生成的分析文件）

将编译好的`tcmalloc.so`复制到对应环境，然后执行

```bash
env LD_PRELOAD="/path_to/libtcmalloc.so" HEAPPROFILE=./analysys.hprof ./your_program
```

具体配置参考[heap profile](https://gperftools.github.io/gperftools/heapprofile.html), 每当目标进程分配 1GB 的内存后，就会生成一份当前进程内存的分析文件（这个可配置的）

随后用`pprof`生成对应的pdf文件，看对应模块的内存分配情况，一般来说如果泄漏的时间够久，那分配最大的地方大概率就是内存泄漏的地方

## 4. bcc

+ [bcc](https://github.com/iovisor/bcc)

本身是一个用于创建高效内核跟踪和操作程序的工具包，基于 eBPF 功能实现。

eBPF在kernel 3.15之后才有，线上要求机器版本。

它最大的优点是非侵入式，同样对进程影响也较小，应该是在条件满足时的最佳使用工具

linux下首推的就是BCC和Asan
