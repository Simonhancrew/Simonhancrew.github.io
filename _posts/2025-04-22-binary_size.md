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


### REF

1. [A Whirlwind Tutorial on Creating Really Teensy ELF Executables for Linux](https://www.muppetlabs.com/~breadbox/software/tiny/teensy.html)
2. [Size optimizations](https://pigweed.dev/docs/size_optimizations.html)
3. [How to make smaller C and C++ binaries](https://news.ycombinator.com/item?id=35853625)
4. [Binary sizes and compiler flags](https://www.sandordargo.com/blog/2023/07/19/binary-sizes-and-compiler-flags)
