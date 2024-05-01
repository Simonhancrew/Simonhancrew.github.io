---
title: Ubuntu18.04下安装高版本gcc并处理版本控制
date: 2023-05-02 14:10:00 +0800
categories: [Blogging]
tags: [writing]
---

## Ubuntu18.04

在这个发行版下默认的安装gcc版本是7.50的，但在做cs144的时候发现他的cmake里写死的gcc支持就在gcc-8以上了，如果不改cmake代码的话，就只能手动安装更高版本的gcc了。详细的gcc关于c++的特性支持可以看[cxx-status](https://gcc.gnu.org/projects/cxx-status.html).

### PPA(Personal Package Archive)

PPA不在Ubuntu默认的软件仓库，每个Ubuntu都有自己的四个官方App Repo

+ Main - Canonical 支持的自由开源软件。
+ Universe - 社区维护的自由开源软件。
+ Restricted - 设备的专有驱动程序。
+ Multiverse - 受版权或法律问题限制的软件。

比如[/ubuntu/dists](http://archive.ubuntu.com/ubuntu/dists/)就可以看到所有发行过的ubuntu版本的所有的仓库。

PPA其实只是一个类似做存储的网址，可以使用PPA源获取到不在官方源里的软件。

这里还有一份详细的[Apt使用指南](https://itsfoss.com/apt-command-guide/), 可以做进一步的阅读。

### 使用PPA并换源

这里以安装更高版本的gcc为例, 12.10以上的系统`add-apt-repository`是由`software-properties-common`提供的，所以需要提前安装， 12.04和以下的系统， 需要安装`python-software-properties`。

``` bash
sudo apt install software-properties-common 
# sudo apt-get install python-software-properties
sudo add-apt-repository ppa:ubuntu-toolchain-r/test
sudo apt update
```
如果要删除当前源, 进入`cd /etc/apt/sources.list.d`删除相应的PPA既可。

```bash
cd /etc/apt/sources.list.d/
```

另外很多PPA的连接稳定性很差，所以换源是十分必要的，比如，针对`ubuntu-toolchain-r-ubuntu-test-bionic.list`, 简单的把`http://ppa.launchpad.net`换成下面的样子就可以了。

```bash
deb https://launchpad.proxy.ustclug.org/ubuntu-toolchain-r/test/ubuntu bionic main
# deb-src https://launchpad.proxy.ustclug.org/ubuntu-toolchain-r/test/ubuntu bionic main
```

### 安装GCC-11并配置版本控制

安装完gcc之后， 可以将其用update-alternatives管理起来,其实就是[符号链接](https://en.wikipedia.org/wiki/Symbolic_link)的管理

```bash
sudo apt install gcc-11 g++-11
sudo update-alternatives --install /usr/bin/gcc gcc /usr/bin/gcc-11 90 --slave /usr/bin/g++ g++ /usr/bin/g++-11
sudo update-alternatives --install /usr/bin/gcc gcc /usr/bin/gcc-7 40 --slave /usr/bin/g++ g++ /usr/bin/g++-7
```

如果要配置gcc的默认版本的话
```bash
sudo update-alternatives --config gcc
```

如果要移除某个版本的话

```bash
# --remove <name> <path>
sudo update-alternatives --remove gcc /usr/bin/gcc-7
```


### Ref

1. [如何在 Ubuntu 18.04 LTS 系统中安装多版本 GCC 编译器](https://www.sysgeek.cn/ubuntu-install-gcc-compiler/#:~:text=%E5%9C%A8Ubuntu%2018.04%E4%B8%AD%E5%AE%89%E8%A3%85GCC%E7%BC%96%E8%AF%91%E5%99%A8%20%E9%BB%98%E8%AE%A4%E7%9A%84%20Ubuntu%20%E5%AD%98%E5%82%A8%E5%BA%93%E4%B8%AD%E5%8C%85%E5%90%AB%E4%B8%80%E4%B8%AA%E5%90%8D%E4%B8%BA%20build-essential%20%E7%9A%84%E8%BD%AF%E4%BB%B6%E5%8C%85%E9%9B%86%E5%90%88%EF%BC%8C%E5%AE%83%E5%8C%85%E5%90%AB%E4%BA%86%20GCC,%E6%82%A8%E5%8F%AA%E9%9C%80%E6%89%A7%E8%A1%8C%E4%BB%A5%E4%B8%8B%E6%AD%A5%E9%AA%A4%E5%B0%B1%E5%8F%AF%E4%BB%A5%E5%9C%A8%20Ubuntu%2018.04%20%E4%B8%AD%E5%AE%89%E8%A3%85%20GCC%20%E7%BC%96%E8%AF%91%E5%99%A8%EF%BC%9A%201%20%E5%9C%A8%E3%80%8C%E7%BB%88%E7%AB%AF%E3%80%8D%E4%B8%AD%E6%89%A7%E8%A1%8C%E4%BB%A5%E4%B8%8B%E5%91%BD%E4%BB%A4%E6%9B%B4%E6%96%B0%E5%8C%85%E5%88%97%E8%A1%A8%EF%BC%9A)

