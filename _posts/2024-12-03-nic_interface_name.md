---
title: linux网卡设备interface name
date: 2024-12-03 16:10:10 +0800
categories: [Blogging, network]
tags: [writing]
---

Linux中有一些标准的接口名称，这里列出了这些名称。大多数驱动程序支持多个接口，在这种情况下，接口会按照eth0和eth1这样的方式编号：

### lo

这是本地环回接口。它用于测试目的，以及一些网络应用程序。它就像一个闭合电路，任何写入的数据报都会立即返回到主机的网络层。内核中始终有一个环回设备存在，拥有多个似乎没有太大意义。

### eth0, eth1, …

这些是Ethernet卡接口。它们用于大多数以太网卡，包括许多并行端口以太网卡。

### tr0, tr1, …

这些是Token Ring卡接口。它们用于大多数Token Ring卡，包括非IBM制造的卡。

### sl0, sl1, …

这些是SLIP接口。SLIP接口按它们为SLIP分配的顺序与串行线路相关联。

### ppp0, ppp1, …

这些是PPP接口。就像SLIP接口一样，一旦将串行线路转换为PPP模式，一个PPP接口就会与其关联。

### plip0, plip1, …

这些是PLIP接口。PLIP通过并行线传输IP数据报。接口在系统启动时由PLIP驱动程序分配，并映射到并行端口上。在2.0.x内核中，设备名称与并行端口的I/O端口之间有直接关系，但在后来的内核中，设备名称是按顺序分配的，就像SLIP和PPP设备一样。

### ax0, ax1, …

这些是AX.25接口。AX.25是业余无线电操作员使用的主要协议。AX.25接口的分配和映射方式与SLIP设备类似。

有许多其他类型的接口可用于其他网络驱动程序。我们只列出了最常见的几个。

### br0, br-lan

通常指的是一种虚拟网络接口（network interface），它可以将多个网络接口（如以太网接口、Wi-Fi接口等）连接到一起，形成一个单一的网络段。桥接接口允许这些不同的网络接口在逻辑上作为一个单一网络的一部分进行通信，就像它们物理性地连接在同一个交换机上一样。

## systemd.net-naming-scheme

实际的前缀有规则可循

 | prefix | Description           |
 | :----- | :-------------------- |
 | en     | Ethernet              |
 | ib     | Infiniband            |
 | sl     | Serial Line IP (SLIP) |
 | wl     | Wireless LAN          |
 | ww     | Wireless WAN          |

## References

1. [A Tour of Linux Network Devices](https://tldp.org/LDP/nag2/x-087-2-hwconfig.tour.html)
2. [systemd.net-naming-scheme — Network device naming schemes](https://www.freedesktop.org/software/systemd/man/latest/systemd.net-naming-scheme.html)