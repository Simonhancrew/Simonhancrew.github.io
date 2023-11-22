---
layout: post
title:  "编译烂活合集"
category: compile
date:   2023-11-22
---

compile上的烂活，可能导致各类的问题

## 在头文件定义宏

在头文件定义宏出现重复定义的其实倒还好，如果一个头文件被include，然后恰好在一个translate unit里面，有一个同名的符号，这个符号会被替换

大多数时候可能还好，但出问题就很恶心，要改头文件顺序。工程大了就是折磨

## 两个静态库依赖同一个静态库导致符号冲突

这个问题在编译器里面是可以解决的，但是在链接器里面就不行了，因为链接器是不知道编译器的信息的

这个问题比较有趣，在[Ref-1](https://selfboot.cn/2023/09/19/c++_symbol_resolution/)里可以详细看看

## Ref

1. [深入理解 C++ 链接符号决议：从符号重定义说起](https://selfboot.cn/2023/09/19/c++_symbol_resolution/)

2. [Library order in static linking](https://eli.thegreenplace.net/2013/07/09/library-order-in-static-linking)
