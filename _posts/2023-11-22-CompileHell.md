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

说实话，这种link导致重复的，gcc下，他发现有一摸一样的符号，可能丢弃第二个库，但是vc就会爆冲突，然后如果用到lib里的某个符号，这个目标里的全部符号都会被导入。

但如果两个库稍有不同，且不同的地方被用到了，两个库就里相关的.o就会完全被导入，结果就是符号会冲突

## 循环依赖

顶级烂活，但是可以处理，但是处理起来很麻烦

## Ref

1. [深入理解 C++ 链接符号决议：从符号重定义说起](https://selfboot.cn/2023/09/19/c++_symbol_resolution/)

2. [Library order in static linking](https://eli.thegreenplace.net/2013/07/09/library-order-in-static-linking)

3. [link order](https://stackoverflow.com/questions/45135/why-does-the-order-in-which-libraries-are-linked-sometimes-cause-errors-in-gcc)
