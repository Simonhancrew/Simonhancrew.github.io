---
title: tensorflow-gpu的安装"
date: 2021-11-19 14:10:00 +0800
categories: [Blogging]
tags: [writing]
---

最近发现tf的安装用anaconda简单了很多,和之前torch一样，不必提前装上cudatoolkit。本地的conda已经换源了。

首先创建虚拟环境

```
conda create -n tf python=3.8
```

然后需要装上cudatoolkit和cudnn，需要对应上实际的tensorflow的版本，此时我要安装的是tf-2.3.0,这两个选择对应上版本就可以了

```
conda install cudatoolkit=10.1 

conda install cudnn=7.6.5
```

最后安装预先选择的tf版本

```
conda install tensorflow-gpu=2.3.0
```

