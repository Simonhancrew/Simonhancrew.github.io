---
title: windows route tables
date: 2024-07-17 21:45:00 +0800
categories: [Blogging, network, route table]
tags: [writing]
---

## 1 route table

默认下，输入`route print`可以查看当前系统的路由表，如下：

```shell
===========================================================================
接口列表
  9...[] ......Realtek PCIe GbE Family Controller
 20...[] ......Intel(R) Wi-Fi 6E AX211 160MHz
 19...[] ......Microsoft Wi-Fi Direct Virtual Adapter
  8...[] ......Microsoft Wi-Fi Direct Virtual Adapter #2
  7...[] ......Fortinet Virtual Ethernet Adapter (NDIS 6.30)
 17...[] ......VMware Virtual Ethernet Adapter for VMnet1
 23...[] ......VMware Virtual Ethernet Adapter for VMnet8
 13...[] ......Bluetooth Device (Personal Area Network)
  1...........................Software Loopback Interface 1
===========================================================================

IPv4 路由表
===========================================================================
活动路由:
网络目标        网络掩码          网关       接口   跃点数
          0.0.0.0          0.0.0.0      192.168.2.1     192.168.2.48     25
        127.0.0.0        255.0.0.0            在链路上         127.0.0.1    331
        127.0.0.1  255.255.255.255            在链路上         127.0.0.1    331
  127.255.255.255  255.255.255.255            在链路上         127.0.0.1    331
      192.168.2.0    255.255.255.0            在链路上      192.168.2.48    281
     192.168.2.48  255.255.255.255            在链路上      192.168.2.48    281
    192.168.2.255  255.255.255.255            在链路上      192.168.2.48    281
     192.168.70.0    255.255.255.0            在链路上      192.168.70.1    291
     192.168.70.1  255.255.255.255            在链路上      192.168.70.1    291
   192.168.70.255  255.255.255.255            在链路上      192.168.70.1    291
    192.168.196.0    255.255.255.0            在链路上     192.168.196.1    291
    192.168.196.1  255.255.255.255            在链路上     192.168.196.1    291
  192.168.196.255  255.255.255.255            在链路上     192.168.196.1    291
        224.0.0.0        240.0.0.0            在链路上         127.0.0.1    331
        224.0.0.0        240.0.0.0            在链路上     192.168.196.1    291
        224.0.0.0        240.0.0.0            在链路上      192.168.70.1    291
        224.0.0.0        240.0.0.0            在链路上      192.168.2.48    281
  255.255.255.255  255.255.255.255            在链路上         127.0.0.1    331
  255.255.255.255  255.255.255.255            在链路上     192.168.196.1    291
  255.255.255.255  255.255.255.255            在链路上      192.168.70.1    291
  255.255.255.255  255.255.255.255            在链路上      192.168.2.48    281
===========================================================================
永久路由:
  无
```

首先看路由表的几个重要的字段：

1. Active Routes ：活动的路由
2. Network destination ： 是网络目的地址。列出了路由器连接的所有的网段。
3. Netmask： 网络掩码列提供这个网段本身的子网掩码，而不是连接到这个网段的网卡的子网掩码。这基本上能够让路由器确定目的网络的地址类。
4. Gateway： 网关。一旦路由器确定它要把这个数据包转发到哪一个目的网络，路由器就要查看网关列表。网关表告诉路由器这个数据包应该转发到哪一个IP地址才能达到目的网络。
5. Interface： 接口列告诉路由器哪一个网卡连接到了合适的目的网络。从技术上说，接口列仅告诉路由器分配给网卡的IP地址。那个网卡把路由器连接到目的网络。然而，路由器很聪明，知道这个地址绑定到哪一个物理网卡。
6. Metric： 跳数，跳数用于指出路由的成本，通常情况下代表到达目标地址所需要经过的跳跃数量，一个跳数代表经过一个路由器。跳数越低，代表路由成本越低，优先级越高。

其次看每个路由表代表什么

1. 第一条信息：缺省路由
   > 缺省路由：意思就是说，当一个数据包的目的网段不在你的路由记录中，那么，你的路由器该把那个数据包发送到那里！缺省路由的网关是由你的连接上的default gateway决定的

路由表的形式一般是：

| 目的网络地址（D） | 子网掩码（M） | 下一跳  (N)  |
| ------------------ | ------------- | ------------ |
| 0.0.0.0            | 0.0.0.0       | 192.168.43.1 |

IP 包如何路由（路由器转发分组）？

1. 从收到的数据报的首部提取目的 IP 地址 D1；
2. 先判断是否为直接交付。对路由器直接相连的网络逐个进行检查：用各网络的子网掩码（M）和 D1 逐位相“与”，看结果是否和相应的网络地址（D）匹配。若匹配，则把分组进行直接交付（当然还需要把 D1 转换成物理地址，把数据报封装成帧发送出去），转发任务结束。否则就是间接交付，执行3；
3. 若路由表中有目的地址为D1的特定主机路由，则把数据报传送给路由表中所指明的下一跳路由器（N）；否则，执行4；
4. 对路由表中的每一行（目的网络地址，子网掩码，下一跳地址），用其中的子网掩码（M）和 D1 逐位相“与”，其结果为 D2。若D2与该行的目的网络地址（D）匹配，则把数据报传送给该行指明的下一跳路由器（N）；否则，执行5；
5. 若路由表中有一个默认路由，则把数据报传送给路由表中所指明的默认路由器；否则，执行6；
6. 转发分组出错。

## 2 配置路由信息

这里主要针对windows，有几种情况需要自己配置默认路由

1. 多网卡，需要走特定的route
2. VPN，需要走特定的route
3. 有多个网关，需要走特定的route

因此有几个命令可以着重看一下：

1. 使用`ipconfig /all`查看网卡信息
2. `route print`看路由信息
3. `route`相关的操作命令

route的命令：

```shell
ROUTE [-f] [-p] [command [destination] [MASK netmask] [gateway] [METRIC metric] [IF interface]
```

1. 其中-f参数用于清除路由表，-p参数用于永久保留某条路由（即在系统重启时不会丢失路由）。
2. Command 主要有 PRINT （打印）、 ADD （添加）、 DELETE （删除）、 CHANGE （修改）共 4 个命令。
3. Destination 代表所要达到的目标 IP 地址。
4. MASK 是子网掩码的关键字。 Netmask 代表具体的子网掩码，如果不加说明，默认是 255.255.255.255 （单机 IP 地址）。如果代表全部出口子网掩码可用 0.0.0.0 。
5. Gateway 代表出口网关。
6. 其他 interface 和 metric 分别代表特殊路由的接口数目和到达目标地址的跳数，一般默认。

比如，我们要删除缺省路由

```shell
route delete 0.0.0.0
```

我们要新增一条路由表

```shell
route add 192.168.5.0 mask 255.255.255.0 192.168.2.254 if 24 -p
```

目标网段是192.168.5.0/24，下一跳是192.168.2.254，作用在接口24上，设置为永久路由 

## REF

1. [路由表配置](https://www.cnblogs.com/Chary/p/13957475.html)
