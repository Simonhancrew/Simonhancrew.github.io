---
title: asan link on mac
date: 2025-07-09 16:53:00 +0800
categories: [Blogging, compile]
tags: [writing]
---

asan的link其实是对编译器版本是有要求的，llvm原生的可能还好，gcc的原生的应该也还行

他们要跨很多个大版本才改一次符号, 现在看起来就很稳, [对应代码](https://github.com/gcc-mirror/gcc/blob/c6ca6e57004653b787d2d6243fe5ee00cda8aad0/libsanitizer/asan/asan_init_version.h#L11)

比如

```
extern "C" {
  // Every time the ASan ABI changes we also change the version number in the
  // __asan_init function name.  Objects built with incompatible ASan ABI
  // versions will not link with run-time.
  //
  // Changes between ABI versions:
  // v1=>v2: added 'module_name' to __asan_global
  // v2=>v3: stack frame description (created by the compiler)
  //         contains the function PC as the 3rd field (see
  //         DescribeAddressIfStack)
  // v3=>v4: added '__asan_global_source_location' to __asan_global
  // v4=>v5: changed the semantics and format of __asan_stack_malloc_ and
  //         __asan_stack_free_ functions
  // v5=>v6: changed the name of the version check symbol
  // v6=>v7: added 'odr_indicator' to __asan_global
  // v7=>v8: added '__asan_(un)register_image_globals' functions for dead
  //         stripping support on Mach-O platforms
#if SANITIZER_WORDSIZE == 32 && SANITIZER_ANDROID
  // v8=>v9: 32-bit Android switched to dynamic shadow
  #define __asan_version_mismatch_check __asan_version_mismatch_check_v9
#else
  #define __asan_version_mismatch_check __asan_version_mismatch_check_v8
#endif
}

#endif  // ASAN_INIT_VERSION_H
```

另外[这份代码](https://github.com/gcc-mirror/gcc/blob/master/libsanitizer/sanitizer_common/sanitizer_linux_libcdep.cpp#L360)里有一个改了结构体导致的bug，但我想应该看不到了


apple的clang是自己拉的分支，所以也有一个version check的符号

```
___asan_version_mismatch_check_apple_clang_xxxx
```

一般这里都是大版本的，所以之前认为在大版本下，小的编译器版本是无所谓的，今天link的时候发现出现undef symbol

```
Undefined symbols for architecture arm64:
"___asan_version_mismatch_check_apple_clang_1300", referenced from:
```

开始针对2个方向排查了

1. 编译器是不是升级了
2. 本地是不是没带ldflags

针对排查1，clang version就够了，另外看了本地也没别的clang version

```
Apple clang version 13.1.6 (clang-1316.0.21.2.5)
```

这里没问题，开始怀疑这个tu是不是没带ldflags, 直接掉gn的config看对应的module，直接走gn ls + gn desc就可以

```
ldflags
  -fsanitize=address
  -fsanitize-address-use-after-scope
```

这里也没问题，暂时无解。只能换方向了，一只没有怀疑apple的符号是不是有变更，只好去看一下本地编译的其他组件是不是对应的符号

nm看一下别的组件的符号

```
___asan_version_mismatch_check_apple_clang_1316
```

GG
