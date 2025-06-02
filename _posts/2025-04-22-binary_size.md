---
title: how to reduce binary size
date: 2025-04-22 10:46:00 +0800
categories: [Blogging, c++, compile]
tags: [writing]
---

### 评估

使用bloaty针对最后交付的动态库或者是binary进行分析

常见的

```bash
armembers       the .o files in a .a file
compileunits    source file for the .o file (translation unit). requires debug info.
inputfiles      the filename specified on the Bloaty command-line
inlines         source line/file where inlined code came from.  requires debug info.
sections        object file section
segments        load commands in the binary
symbols         symbols from symbol table (configure demangling with --demangle)
rawsymbols      unmangled symbols
fullsymbols     full demangled symbols
shortsymbols    short demangled symbols 
```

一般都是要看那个文件大，然后去分析, 比如就看bloaty的话

```bash
./bloaty ./bloaty -d compileunits -n 0

    FILE SIZE        VM SIZE
 --------------  --------------
```

file size是文件大小，vm size是到虚拟内存的大小，会看到每个compile unit的大小，这样就能花更多的时间在具体文件的体积分析上

### 具体的几个可行方法

首先我们的优化都是在开启release模式下，objcopy之类方法分离掉了调试符号之后，针对最终的产物进行优化

同时我们已经完全禁用了异常 + RTTI，这两个禁用也能减少很多的包体积

#### 1 根据bloaty的结果，裁剪具体的文件

最终产物是so的话，理论上我用了dead strip之类的东西，但是最后看产物的时候还是会有非预期的符号，这里暂时当作dead_strip的算法不聪明，也可能是姿势不对之类的。

所以这里需要做两件事

1. 我不确定能不能手动移除符号，但是这个工程量太多了
2. 我可以通过编译器的选项来移除符号，通过宏来隔离代码，但是这个要小心，因为一定会有bug的，比如我就裁剪多了

#### 2 上编译器选项

##### 2.1 使用`-Os`来优化代码大小

主要是优化级别了，这里直接查查就可以了

##### 2.2 使用LTO

另外还有LTO之类的也有帮助，但是来的不多，这里还是之前那个dead strip的问题，感觉她不够聪明，deadcode没完全剔除，同时我们这里已经是ThinLTO了，速度感觉上也不慢。这里就没做过多优化了

另外比较搞的是我们有gcc + clang混编的

1. 如果使用 Clang，编译参数和链接参数中都要开启 LTO，否则会出现无法识别文件格式的问题（NDK22 之前存在此问题）。使用 GCC 的话，只需要编译参数中开启 LTO 即可。
2. 我们有很多complete lib，即二进制.a交付的静态库，这个也要一起开启LTO才能work，但是这个工作量就有点大了，每个元编译系统都不一样，我能看到的就有
   1. CMake
   2. Bazel
   3. GN
3. ld也要开优化选项

#### 2.3 gcc的gc section + clang的dead strip

另外还带上Garbage Collection的选项，这个参考美团那个实践就可以了，他在这里讲到了一点我比较感兴趣`编译器默认会把所有函数放到同一个 section 中，把所有相同特点的数据放到同一个 section 中，如果同一个 section 中既有需要删除的部分又有需要保留的部分，会使得整个 section 都要保留。所以我们需要减小目标文件 section 的粒度`。这个跟链接有点像，只能整个o的导入，没法分割符号。

```cmake
# Gcc
set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -fdata-sections -ffunction-sections")
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fdata-sections -ffunction-sections")
set(CMAKE_SHARED_LINKER_FLAGS "${CMAKE_SHARED_LINKER_FLAGS} -Wl,--gc-sections")
```

clang就比较简单，不带gc-sections，带dead_strip就可以了

#### 3 ld script

这个就是动态导出script配合visibility的使用了，我们用clang，直接`-fvisibility=hidden`, 配合要导出的符号使用`__attribute__((visibility("default")))`来标记


### 总结

dead strip没work的那么好，我们别的优化也早就做到工程里了。

也可能是裁剪的姿势问题，最后还是靠分析那些可以不要的符号来裁剪的

这个快，但是风险也大，要对代码动刀，最后效果也非常的明显，优化了一半。

### REF

1. [A Whirlwind Tutorial on Creating Really Teensy ELF Executables for Linux](https://www.muppetlabs.com/~breadbox/software/tiny/teensy.html)
2. [Size optimizations](https://pigweed.dev/docs/size_optimizations.html)
3. [How to make smaller C and C++ binaries](https://news.ycombinator.com/item?id=35853625)
4. [Binary sizes and compiler flags](https://www.sandordargo.com/blog/2023/07/19/binary-sizes-and-compiler-flags)
5. [Android对so体积优化的探索与实践](https://tech.meituan.com/2022/06/02/meituans-technical-exploration-and-practice-of-android-so-volume-optimization.html)
