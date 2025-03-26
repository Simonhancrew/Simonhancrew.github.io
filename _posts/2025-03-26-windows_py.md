---
title: windows原生python多环境pylauncher管理
date: 2025-03-26 21:46:00 +0800
categories: [Blogging, win, python3]
tags: [writing]
---

windows平台下，python官方包会安装[pylauncher](https://docs.python.org/3/using/windows.html#python-launcher-for-windows)，用于管理多个python环境。

通过`py --list`就可以看到安装了多少个python的版本

```shell
$ py --list
 -V:3.13 *        Python 3.13 (64-bit)
 -V:3.7           Python 3.7 (64-bit)
 -V:2.7           Python 2.7
```

### 使用指定版本的python

比如要指定使用python3.7，可以使用`py -3.7`, 剩下的同理

```shell
$ py -3.7
```

### 使用指定的预设python版本

当前默认的py是`py--list`显示的标星号的版本，默认会用最新的python版本

```shell
py --version

Python 3.13.2
```

#### 永久修改默认版本

需要修改当前版本的话，可以通过在LocalAppData目录创建一个py.ini文件，内容如下

```ini
[defaults]
python=3.7
```

再跑`py --version`就会显示3.7了


#### 临时shell中修改默认版本

```shell
$env:PY_PYTHON3 = '3.7'
```

这里其实不区分大小写，当前shell中就会临时使用3.7版本作为默认的py

### 修改主版本默认的python环境

`py -3`默认会使用最新的python3版本，如果想要使用3.7，可以通过`py -3.7`

如果想要修改`py -3`指定的python版本, python2是类似的

```shell
$env:py_python3 = '3.7'
```

此时再跑`py -3`就会使用3.7版本的python了

```
# py -3 --version
Python 3.13.2
# $env:py_python3=3.7
# py -3 --version
Python 3.7.3
```

pylauncher的管理只能在win下使用，不支持跨平台

如果有更高的要求，可以考虑

1. [pyenv](https://github.com/pyenv/pyenv/)
2. [uv](https://github.com/astral-sh/uv)
