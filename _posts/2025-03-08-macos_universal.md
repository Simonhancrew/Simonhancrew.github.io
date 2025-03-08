---
title: 简单的编译macos universal二进制
date: 2025-03-08 13:27:00 +0800
categories: [Blogging, compile]
tags: [writing]
---

今天看的时候发现只要在cxx + link的时候带上

```
arch -x86_64 arch -arm64
```

翻译过来的话就是ninja规则类似

```
rule cxx
  command = $cxx -MMD -MF $out.d $includes $cflags -arch x86_64 -arch arm64 -c $in -o $out
  description = CXX $out
  depfile = $out.d
  deps = gcc

rule alink_thin
  command = rm -f $out && $ar rcs $out $in
  description = AR $out

rule link
  command = $ld $ldflags -arch x86_64 -arch arm64 -o $out $in $solibs $libs
  description = LINK $out
```

cmake可以在编译的时候带上CMAKE_OSX_ARCHITECTURES

```
cmake -DCMAKE_OSX_ARCHITECTURES="x86_64;arm64" ..
```

还有就是可以分开编译然后用lipo合并

```
lipo -create x86_64.bundle arm64.bundle -output universal.bundle
```
