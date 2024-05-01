---
title: 编译烂活合集"
date: 2023-11-22 14:10:00 +0800
categories: [Blogging, compile]
tags: [writing]
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

很多人一直觉得编译的知识不是很重要，只要会用cmake之类的工具就行了，虽然确实也是这样，但我更愿意知其所以然

[Ref2](#ref)和[Ref4](#ref)里有详细的关于链接处理的内容

## 循环依赖

顶级烂活，但是可以处理，但是处理起来很麻烦，强连通，要链接几次。简单点的办法就是start group和end group

## debug + release库混用

这个在win下存在比较骚的东西，但我之前比较少遇到(因为我之前也debug用release的库，或者release用debug的库)

[container-flags](https://learn.microsoft.com/en-us/cpp/standard-library/checked-iterators?view=msvc-170)

这里面讲了一个flag，在win下link的时候如果flag不匹配会报错。一般办错理由是

```
error LNK2038: mismatch detected for '_ITERATOR_DEBUG_LEVEL': value '0' doesn't match value '2' in *.lib
```

这个时候找到混用了的库就行。

## 混用不同版本的库在不同的组件中

[杂谈：一个 C++ Header-only 版本冲突的案例分析](https://zhuanlan.zhihu.com/p/684965383)

主要看看上面这个，增加点记性，我发现我司的也很多这种问题，只不过现在没爆雷

## Ref

1. [深入理解 C++ 链接符号决议：从符号重定义说起](https://selfboot.cn/2023/09/19/c++_symbol_resolution/)

2. [Library order in static linking](https://eli.thegreenplace.net/2013/07/09/library-order-in-static-linking)

3. [link order](https://stackoverflow.com/questions/45135/why-does-the-order-in-which-libraries-are-linked-sometimes-cause-errors-in-gcc)

4. [How statically linked programs run on Linux](https://eli.thegreenplace.net/tag/linkers-and-loaders)

5. [Load-time relocation of shared libraries](https://eli.thegreenplace.net/2011/08/25/load-time-relocation-of-shared-libraries)

6. [Position Independent Code (PIC) in shared libraries](https://eli.thegreenplace.net/2011/11/03/position-independent-code-pic-in-shared-libraries)

7. [Position Independent Code (PIC) in shared libraries on x64](https://eli.thegreenplace.net/2011/11/11/position-independent-code-pic-in-shared-libraries-on-x64)

8. [Why can’t I __declspec(dllexport) a function from a static library](https://devblogs.microsoft.com/oldnewthing/20140321-00/?p=1433)

9. [Understanding the classical model for linking, groundwork: The algorithm](https://devblogs.microsoft.com/oldnewthing/20130107-00/?p=5633)

10. [Understanding the classical model for linking: Taking symbols along for the ride](https://devblogs.microsoft.com/oldnewthing/20130108-00/?p=5623)

11. [Understanding the classical model for linking: You can override an LIB with another LIB, and a LIB with an OBJ, but you can’t override an OBJ
](https://devblogs.microsoft.com/oldnewthing/20130109-00/?p=5613)

12. [Understanding the classical model for linking: Sometimes you don’t want a symbol to come along for a ride](https://devblogs.microsoft.com/oldnewthing/20130110-00/?p=5593)

13. [Understanding errors in classical linking: The delay-load catch-22
](https://devblogs.microsoft.com/oldnewthing/20130111-00/?p=5583)

14. [Why does the Windows Portable Executable (PE) format have separate tables for import names and import addresses](https://devblogs.microsoft.com/oldnewthing/20231129-00/?p=109077)

15. [Why does the Windows Portable Executable (PE) format have both an import section and input directory?
](https://devblogs.microsoft.com/oldnewthing/20231201-17/?p=109090)
