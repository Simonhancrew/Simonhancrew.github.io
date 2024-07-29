---
title: 交叉编译
date: 2024-07-25 23:34:00 +0800
categories: [Blogging, compile, cross compile]
tags: [writing]
---

简单看下cross compile要做哪些东西，从一个gn的例子开始。

```txt
config("compiler_cpu_abi") {
  cflags = []
  cflags_c = []
  cflags_cc = []
  ldflags = []
  defines = []

  if ((is_android || is_linux || is_hmos) && !(is_mac || is_ios)) {
    # CPU architecture. We may or may not be doing a cross compile now, so for
    # simplicity we always explicitly set the architecture.
    if (current_cpu == "x64") {
      cflags += [
        "-m64",
        "-march=x86-64",
      ]
      ldflags += [ "-m64" ]
    } else if (current_cpu == "x86") {
      cflags += [ "-m32" ]
      ldflags += [ "-m32" ]

      cflags += [
        "-msse2",
        "-mfpmath=sse",
        "-mmmx",
      ]
    } else if (current_cpu == "arm") {
      if (is_clang && !is_android && !is_hmos) {
        cflags += [
          "--target=arm-linux-gnueabihf",
          "-mthumb",
        ]
        ldflags += [ "--target=arm-linux-gnueabihf" ]
      }

      if (is_android) {
        cflags += [ "-march=armv7-a" ]
      }

      if (arm_float_abi == "") {
        if (current_os == "android") {
          arm_float_abi = "softfp"
        } else if (target_os == "android") {
          arm_float_abi = "softfp"
        } else if (current_os == "linux") {
          arm_float_abi = "hard"
        } else {
          arm_float_abi = "hard"
        }
      }

      cflags += [ "-mfloat-abi=$arm_float_abi" ]
    } else if (current_cpu == "arm64") {
      if (is_clang && !is_android) {
        if(is_hmos){
        cflags += [ "--target=aarch64-linux-ohos" ]
        ldflags += [ "--target=aarch64-linux-ohos" ]
        } else {
        cflags += [ "--target=aarch64-linux-gnu" ]
        ldflags += [ "--target=aarch64-linux-gnu" ]
        }
      }
    }
  }

  if (is_linux && target_cpu != host_cpu &&
      !(target_cpu == "x86" && host_cpu == "x64")) {
    # Needs additional sysroot
    sysroot = getenv("SYSROOT")
    cpp_headers = getenv("CPP_HEADERS")
    assert(sysroot != "", "Must provide SYSROOT environment variable")
    if (cpp_headers != "") {
       cflags += [ "-isystem$cpp_headers" ]
    }
    cflags += [ "--sysroot=$sysroot" ]
    ldflags += [ "--sysroot=$sysroot" ]
  }

  asmflags = cflags
}
```

著平台的解释一下这个cpu-abi

## linux

## win

## darwin

## android

## REF

1. [gcc交叉编译时设置了“--sysroot“会产生哪些影响](https://blog.csdn.net/zvvzxzko2006/article/details/110467542)
