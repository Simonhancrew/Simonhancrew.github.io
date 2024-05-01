---
title: dynamic load dll"
date: 2024-01-11 14:10:00 +0800
categories: [Blogging, compile]
tags: [writing]
---

动态库热加载指的是在程序运行时，动态地加载动态库，从而达到不停止程序的情况下，更新程序的功能。

C++ 程序在运行时有两种方式加载动态连接库：隐式链接和显式链接

1. 隐式链接就是在编译的时候使用 -l 参数链接的动态库，进程在开始执行时就将动态库文件映射到内存空间中

2. 显式链接使用`libdl.so`库的`API`接口在运行中加载和卸载动态库，主要的`API`有 `dlopen`、`dlclose`、`dlsym`、`dlerror`。

### REF

1. [C++ 动态库热加载](https://zhuanlan.zhihu.com/p/676040808)

2. [Build a Live Code-reloader Library for C++](https://howistart.org/posts/cpp/1/index.html)
