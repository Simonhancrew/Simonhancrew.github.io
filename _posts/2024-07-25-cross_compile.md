---
title: 交叉编译
date: 2024-07-25 23:34:00 +0800
categories: [Blogging, compile, cross compile]
tags: [writing]
---

简单看下cross compile要做哪些东西，从一个gn的例子开始。

```python
config("compiler_cpu_abi") {
  cflags = []
  cflags_c = []
  cflags_cc = []
  ldflags = []
  defines = []

  if (is_linux) {
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

## linux下的交叉编译

首先指定架构

### x64

```python
if (current_cpu == "x64") {
  cflags += ["-m64", "-march=x86-64"]
  ldflags += ["-m64"]
}
```

+ -m64: 指定生成 64 位代码
+ -march=x86-64: 指定目标架构为 x86-64

### x32

```python
else if (current_cpu == "x86") {
  cflags += ["-m32", "-msse2", "-mfpmath=sse", "-mmmx"]
  ldflags += ["-m32"]
}
```

+ -m32: 指定生成 32 位代码
+ -msse2, -mfpmath=sse, -mmmx: 启用特定的 x86 指令集扩展

### arm

```python
else if (current_cpu == "arm") {
  if (is_clang && !is_android && !is_hmos) {
    cflags += ["--target=arm-linux-gnueabihf", "-mthumb"]
    ldflags += ["--target=arm-linux-gnueabihf"]
  }
  // ... (其他 ARM 特定设置)
}
```

+ --target=arm-linux-gnueabihf: 指定目标为 ARM Linux，使用硬浮点 ABI
+ -mthumb: 使用 Thumb 指令集
+ 根据不同的操作系统（如 Android）设置不同的浮点 ABI

### arm64

```python
else if (current_cpu == "arm64") {
  if (is_clang && !is_android) {
    cflags += ["--target=aarch64-linux-gnu"]
    ldflags += ["--target=aarch64-linux-gnu"]
  }
}
```

### 指定sys_root + cpp header路径

```python
if (is_linux && target_cpu != host_cpu &&
    !(target_cpu == "x86" && host_cpu == "x64")) {
  sysroot = getenv("SYSROOT")
  cpp_headers = getenv("CPP_HEADERS")
  // ... (设置 sysroot 和 C++ 头文件路径)
}
```

+ 当目标 CPU 与主机 CPU 不同时，设置交叉编译的 sysroot 和头文件路径
+ 使用 --sysroot 选项指定交叉编译的根文件系统路径

## cmake如何写

```cmake
set(CMAKE_SYSTEM_NAME Linux)
set(CMAKE_SYSTEM_VERSION 1)
set(CMAKE_SYSTEM_PROCESSOR aarch64)

set(SYSROOT_PATH  /path/to/sysroot)
set(CMAKE_SYSROOT "${SYSROOT_PATH}")
message(STATUS  "Using sysroot path as ${SYSROOT_PATH}")

set(CMAKE_STAGING_PREFIX /path/to/staging/)
set(CMAKE_INSTALL_PREFIX /usr/local)

set(TOOLCHAIN_PATH /path/to/toolchain)
set(TOOLCHAIN_HOST ${TOOLCHAIN_PATH}/bin/aarch64-linux-gnu)

set(TOOLCHAIN_CC "${TOOLCHAIN_HOST}-gcc")
set(TOOLCHAIN_CXX "${TOOLCHAIN_HOST}-g++")

set(CMAKE_C_COMPILER ${TOOLCHAIN_CC})
set(CMAKE_CXX_COMPILER ${TOOLCHAIN_CXX})

add_link_options("LINKER:-rpath-link,/path/to/sysroot/lib/aarch64-linux-gnu:/home/admin/tx2-rootfs/usr/lib/aarch64-linux-gnu")

set(CMAKE_FIND_ROOT_PATH_MODE_PROGRAM NEVER)
set(CMAKE_FIND_ROOT_PATH_MODE_LIBRARY ONLY)
set(CMAKE_FIND_ROOT_PATH_MODE_INCLUDE ONLY)
set(CMAKE_FIND_ROOT_PATH_MODE_PACKAGE ONLY)
```

### 总结

一般来讲，clang的跨平台编译不用太过操心，你只需要设置好target。

在linux下，还需要设置好sysroot，如果你还要自己指定cpp header路路径，你需要继续设置-isystem选项。

针对gcc的cross compile，你可能还需要用不同的cc和cxx来指定编译器，同时还需要设置好sysroot。

sysroot的官方介绍中

```ascii
--sysroot=dir
Use dir as the logical root directory for headers and libraries. For example, if the compiler normally searches for headers in /usr/include and libraries in /usr/lib, it instead searches dir/usr/include and dir/usr/lib.

If you use both this option and the -isysroot option, then the --sysroot option applies to libraries, but the -isysroot option applies to header files.

The GNU linker (beginning with version 2.16) has the necessary support for this option. If your linker does not support this option, the header file aspect of --sysroot still works, but the library aspect does not.
```

所以理论上你要交叉编译的话，sysroot里只要有/usr/include和/usr/lib就可以了。

但是也不一定，我见过比较怪的工具链，aarch下只有lib + lib64 + include, 没有usr/, 这种情况下你就需要自己指定cpp header路径了

## REF

1. [gcc交叉编译时设置了“--sysroot“会产生哪些影响](https://blog.csdn.net/zvvzxzko2006/article/details/110467542)
