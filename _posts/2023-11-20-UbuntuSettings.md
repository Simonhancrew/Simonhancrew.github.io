---
title: Ubuntu Settings
date: 2023-11-20 14:10:00 +0800
categories: [Blogging, tools]
tags: [writing]
---

ubuntu开箱即用设置

## Ubuntu18.04

主要做3件事

1. 换源

2. 装必须的软件

3. 个人使用环境相关的初始化，比如git，vim，zsh等

这三者都可以通过一个脚本来完成，另外ubuntu18.04默认的compile-tools-chain实在是太老了，gcc7又是相当垃圾的一个版本，所以需要用ppa的源来安装稍微新一点点的clang和gcc

### Ref

1. [gpg: keyserver receive failed: Server indicated a failure 解决](https://einverne.github.io/post/2019/09/gpg-keyserver-receive-failed-server-indicated-a-failure.html)

2. [清华mirror-ubuntu-换源-help](https://mirrors.tuna.tsinghua.edu.cn/help/ubuntu/)

3. [python2 get pip](https://bootstrap.pypa.io/pip/2.7/get-pip.py)

4. [Ubuntu下安装高版clang](https://codeantenna.com/a/ioXufiRy3V)
