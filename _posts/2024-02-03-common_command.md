---
title: quick commands
date: 2024-02-03 14:10:00 +0800
categories: [Blogging, tools]
tags: [writing]
---

记录一些常用的命令

## 1 nc

### 1 建立TCP连接

`nc hostname port`

### 2 监听TCP端口

`nc -l -p port`

监听UDP的命令是`nc -u -l -p port`

### 3 发送文件

`nc -w 3 hostname port < file`

### 4 接收文件

`nc -l -p port > file`

### 5 发送UDP数据

`echo "data" | nc -u -w 1 hostname port`

### 6 扫描开放的端口

`nc -zv hostname start_port-end_port`

### 7 通过HTTP请求发送数据

`echo -e "GET / HTTP/1.1\nHost: hostname\n\n" | nc hostname port`

### 8 创建反向shell： 在目标机器上运行

`nc -l -p port -e /bin/bash`

## 2 lsof

list open files

### 1 查看端口占用情况

`lsof -i :port`

### 2 查看某个文件被哪些进程占用

`lsof filename`

### 3 查看某个进程打开了哪些文件

`lsof -p pid`

## 3 ps

process status 

### 1. 查看所有进程

`ps -ef`或者`ps aux`

“e” 表示显示环境变量，"f" 表示显示完整格式，"a" 表示显示所有用户的进程，"u" 表示显示用户/所有者信息，"x" 表示显示没有控制终端的进程。

### 2. 查看运行中的进程

`ps -e`

### 3. 查看某个用户的进程

`ps -u username`

### 4. 树状图显示

`ps axjf`

## 4 ss

+ `ss`：如果没有任何参数，ss 会列出开放的非监听 TCP 套接字。

+ `ss -t`：列出所有 TCP 套接字。

+ `ss -u`：列出所有 UDP 套接字。

+ `ss -l`：列出所有监听的套接字。

+ `ss -a`：列出所有套接字（包括监听和非监听的）。

+ `ss -n`：不解析服务名称，直接显示端口号。

+ `ss -p`：显示进程使用的套接字。

+ `ss -e`：显示详细的套接字信息。

+ `ss -o`：显示 timer 信息.

## 5 ip

+ 查看网络设备信息：使用 `ip link show` 或者简写为 `ip l`，这将显示所有网络设备的信息，包括设备名称、状态（UP 表示启用，DOWN 表示关闭）、MAC 地址等。

+ 启动或关闭网络设备：使用 `ip link set dev DEVICE up` 或 `ip link set dev DEVICE down`，其中 DEVICE 是你要操作的设备的名称。例如，要启动名为 eth0 的设备，你可以输入 `ip link set dev eth0 up`。

+ 查看 IP 地址：使用 `ip addr show` 或者简写为 `ip a`，这将显示所有网络设备的 IP 地址信息。

+ 添加或删除 IP 地址：使用 `ip addr add IP dev DEVICE` 或 `ip addr del IP dev DEVICE`，其中 IP 是你要添加或删除的 IP 地址，DEVICE 是你要操作的设备的名称。例如，要给 eth0 设备添加 IP 地址 192.168.1.2，你可以输入 `ip addr add 192.168.1.2 dev eth0`。

+ 查看路由信息：使用 `ip route show` 或者简写为 `ip r`，这将显示所有的路由信息。

+ 添加或删除路由：使用 `ip route add NETWORK via GATEWAY` 或 `ip route del NETWORK via GATEWAY`，其中 NETWORK 是你要添加或删除的网络，GATEWAY 是你要设置的网关。例如，要添加一个到 192.168.1.0/24 网络的路由，网关为 192.168.1.1，你可以输入 `ip route add 192.168.1.0/24 via 192.168.1.1`。
