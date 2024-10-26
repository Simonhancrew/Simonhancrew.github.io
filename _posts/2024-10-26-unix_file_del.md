---
title: unix下删除文件报错处理
date: 2024-10-26 16:40:10 +0800
categories: [Blogging, unix]
tags: [writing]
---

描述一下删除文件可能遇到的错误场景

最常见的情况就是文件属性问题，比如权限不够，或者文件被保护之类的

```bash
lsattr -a <file>
chattr -R -a -i <file>
rm -rf <file>
```

用lsattr看文件属性，常见的描述

```ascii

a - append only
c - compressed
d - no  dump
e - extent  format
i -  immutable
j - data journalling
s - secure deletion
t - no tail-merging
u - undeletable
A - no  atime  updates
D - synchronous directory updates
S - synchronous updates
T - top  of  directory  hierarchy
```

描述一下常见的错误情况和处理方法

### 1. Directory not empty

```bash
rm -rf
```

### 2. Permission denied

```bash
sudo rm
#or
sudo chmod +w path
```

### 3. Device or resource busy

```bash
rm: cannot remove 'directory': Device or resource busy
```

此时文件正在被占用或者挂载，用lsof或者fuser检查占用，

```bash
lsof -D directory
fuser -vm directory
```

如果是挂载的文件，umount之后再删除

```bash
umount /mnt
```

### 4. read-only file system

```bash
rm: cannot remove 'file': Read-only file system
```

将文件系统挂载为可写

```bash
mount | grep path_mound
sudo mount -o remount,rw path_mound
```

### 5. Operation not permitted

```bash
rm: cannot remove 'file': Operation not permitted
```

此时就回头文章开头的方法

```bash
lsattr -a <file>
chattr -R -a -i <file>
rm -rf <file>
```

### 6. Too many levels of symbolic links

符号链接循环, 这个时候就要手动查下了

### 7. File name too long

```bash
rm: cannot remove 'file': File name too long
```

这种使用相对路径或者缩短路径名

### 8. Not a directory

```bash
rmdir: cannot remove 'file': Not a directory
```

使用rm删除文件
