---
title: linux下qemu调试内核
date: 2026-05-30 12:30:00 +0800
categories: [Blogging, linux, kernel, debugging]
tags: [linux, qemu, kernel-debugging, gdb, rootfs]
mermaid: true
---

Ubuntu 24.04 环境下, 调试 Linux 内核, 之前也写过，但是发现新的版本的工具链和内核版本有一些变化，所以重新整理一下。

### 第一阶段：环境与依赖准备

在开始编译之前，确保所有必要工具已就绪：
```bash
# 更新并安装必要的编译和调试依赖
sudo apt update
sudo apt install build-essential libncurses-dev bison flex libssl-dev libelf-dev \
                 libdw-dev dwarves gdb qemu-system-x86 busybox-static
```

### 第二阶段：内核配置与编译

下载代码去kernel.org，选择一个稳定版本（如 7.x）：

之后按照流程进行编译：

1. **解压代码**：
   ```bash
   xz -d linux-x.x.x.tar.xz
   tar -xvf linux-x.x.x.tar
   cd linux-x.x.x
   ```
2. **清理环境**（如果之前编译失败过）：
   ```bash
   make mrproper
   ```
3. **加载默认配置并开启调试选项**：
   ```bash
   make defconfig
   make menuconfig
   ```
   在菜单中务必开启：
   * `Kernel hacking` -> `Compile-time checks` -> `Compile the kernel with debug info` (勾选)
   * `Kernel hacking` -> `Generic Kernel Debugging Instruments` -> `Provide GDB scripts` (勾选)
   * `Processor type` -> `Randomize the address of the kernel image (KASLR)` (**取消勾选**)
   这些分别是
   1. 告诉编译器（GCC）在编译时生成 DWARF（Debugging With Arbitrary Record Formats） 格式的调试信息。
   2. 在内核编译输出目录中生成一系列用于 GDB 的辅助脚本（Python 编写的）。
   3. 如果你开启了 KASLR，每次重启 QEMU，内核加载的物理基地址都在变，关闭了固定内核的内存偏移量。这样你可以在 GDB 中设置固定的断点，甚至编写简单的调试脚本，而不必每次重启都重新计算偏移地址。
   
4. **编译内核**：
   ```bash
   make -j$(nproc)
   ```
   *编译成功后，内核镜像位于 `arch/x86/boot/bzImage`*。

   如果出现了证书报错
   ```bash
   scripts/config --set-str SYSTEM_TRUSTED_KEYS ""
   scripts/config --set-str SYSTEM_REVOCATION_KEYS ""
   ```

### 第三阶段：制作根文件系统 (Rootfs)

为了让内核能够启动并提供交互环境：

1. **构建目录**：
   ```bash
   mkdir -p ~/rootfs/{bin,dev,etc,proc,sys}
   cd ~/rootfs
   cp /bin/busybox bin/
   cd bin
   ./busybox --install -s
   ```
2. **编写 `init` 脚本**（在 `~/rootfs` 下新建 `init` 文件）：
   ```bash
   #!/bin/busybox sh
   mount -t proc proc /proc
   mount -t sysfs sysfs /sys
   echo "System booted successfully!"
   /bin/sh
   ```
   给它执行权限：`chmod +x init`
3. **打包**：
   ```bash
   find . -print0 | cpio --null -ov --format=newc | gzip -9 > ../rootfs.cpio.gz
   ```

### 第四阶段：启动与调试

1. **启动 QEMU**：
   建议创建一个启动脚本 `start_qemu.sh`：
   ```bash
   qemu-system-x86_64 \
       -kernel arch/x86/boot/bzImage \
       -initrd ../rootfs.cpio.gz \
       -append "console=ttyS0 nokaslr" \
       -nographic \
       -s -S
   ```
2. **启动 GDB 调试**：
   打开一个新的终端，进入内核源码目录：
   ```bash
   gdb vmlinux
   ```
   在 GDB 提示符下：
   ```gdb
   (gdb) target remote :1234
   (gdb) break start_kernel
   (gdb) continue
   ```

### 调试小贴士：

* **GDB 自动加载脚本**：当你连接后，GDB 可能会提示你“加载自动加载脚本以获取内核辅助功能”。在你的 `~/.gdbinit` 文件中添加一行 `add-auto-load-safe-path /path/to/your/linux-source/`，这样可以使用 `lx-lsmod` 等内核专用调试命令。

* **修改内核后**：如果修改了代码，只需重新运行 `make -j$(nproc)`，然后重启 QEMU 即可，不需要重新制作 `rootfs`。

* **退出 QEMU**：如果使用 `-nographic` 模式，按下 `Ctrl+A` 然后按 `X` 即可强行退出 QEMU。
