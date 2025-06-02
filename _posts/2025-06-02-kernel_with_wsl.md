---
title: wsl2下kernel编译
date: 2025-06-02 12:07:00 +0800
categories: [Blogging, linux, kernel]
tags: [writing]
---

kernel的编译，在wsl2下。 另外因为我要用qemu调试，所以步骤可能有一丢丢特殊

### 下载kernel

国内直接从阿里镜像下的，试过官方那个cdn的，基本跑不满

+ [阿里云linux-v6.x镜像](https://mirrors.aliyun.com/linux-kernel/v6.x/)

我这边直接用最新的了

### 解压 + 安装必须的依赖

下的gz的包，直接`tar xzf`解压

```bash
sudo apt-get install git fakeroot build-essential ncurses-dev xz-utils libssl-dev bc flex libelf-dev bison
```

这些玩意有什么作用

| Package         | Description                                                                                                           |
| --------------- | --------------------------------------------------------------------------------------------------------------------- |
| git             | Tracks and makes a record of all changes during development in the source code. It also allows reverting the changes. |
| fakeroot        | Creates a fake root environment.                                                                                      |
| build-essential | Installs development tools such as C, C++, gcc, and g++.                                                              |
| ncurses-dev     | Provides API for text-based terminals.                                                                                |
| xz-utils        | Provides fast file compression and decompression.                                                                     |
| libssl-dev      | Supports SSL and TLS to encrypt data and secure internet connections.                                                 |
| bc              | Supports the interactive execution of statements (Basic Calculator).                                                  |
| flex            | Generates lexical analyzers that convert characters into tokens.                                                      |
| libelf-dev      | Provides a shared library for managing ELF files (executables, core dumps, object code).                              |
| bison           | Converts grammar descriptions to C programs.                                                                          |

另外，因为要使用qemu，所以需要额外的安装qemu相关的东西

```bash
sudo apt-get install qemu
sudo apt-get install qemu-system
```

这个版本是真的不高，后面可以自行编译一个更高版本的qemu

```bash
qemu-io version 5.2.0 (Debian 1:5.2+dfsg-11+deb11u4)
Copyright (c) 2003-2020 Fabrice Bellard and the QEMU Project developers
```

### 指定编译架构配置kernel

使用qemu的方式来编译kernel，必须指定架构

```bash
export ARCH=x86
make x86_64_defconfig
make menuconfig
```

如果不指定架构的话

```bash
cp -v /boot/config-$(uname -r) .config
make menuconfig
```

### menuconfig配置

这里主要要做的是让kernel带调试信息的编译

### 编译内核

```bash
make -j$(nproc)
```

### 下载 + 配置busybox + 编译

busybox是一个非常小巧的工具集，提供了很多常用的命令行工具。

[下载链接](https://busybox.net/downloads/)

### 制作rootfs

rootfs是Linux系统中最基本的文件系统，是其他所有文件系统的挂载点

### 启动qemu调试

qemu-system-x86_64的启动命令就可以了，这个看版本找最佳实践就可以了

### compile commands生成

这一步要做编译之后做

```bash
python3 scripts/clang-tools/gen_compile_commands.py
```

另外这个有gcc专有的compile flags，会影响clangd的解析

+ -mindirect-branch
+ -mfunction-return

直接处理一下就可以了

## REF

1. [WSL2之QEMU安装与使用](https://blog.csdn.net/fangye945a/article/details/121962409)
2. [Debug Linux kernel on WSL2 with VScode](https://zhuanlan.zhihu.com/p/652682080)
3. [Linux Kernel – How to setup your editor](https://devmp.org/linux/kernel/2020/12/04/Linux-Kernel-Setup-Your-Editor.html)
