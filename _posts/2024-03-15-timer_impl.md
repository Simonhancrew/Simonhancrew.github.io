---
layout: post
title:  "定时器实现"
category: os
date:   2024-03-15
---

了解一下通用的定时器实现, 大多数的通用库是实现其实都不是高精度的timer。我大概看了些，基本的实现都是timer fd + 一个红黑树存储（或者堆/跳表）

过期时间戳，然后用epoll来监听timer fd的读事件，然后在epoll的回调函数里面一次性处理过期的定时器。但是怎么设计task + loop + timer这一套东西，每个库都有自己的一套，核心其实都是这玩意

部分库实现了取消定时器的能力，但是看起来这玩意在win下的实现就比较麻烦了。

## REF

1. [workflow的timer](https://zhuanlan.zhihu.com/p/665046758)
