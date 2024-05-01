---
title: memory barrier
date: 2024-01-10 14:10:00 +0800
categories: [Blogging, os, memory]
tags: [writing]
---

主要资料是mem barrier，在ref-1。 如果完整看过perfbook的话，不用再看了，paul真的是神人，RCU的作者，修了linux内核1000多个多线程bug的猛男


### REF

1. [Memory Barriers: a Hardware View for Software Hackers](http://www.rdrop.com/users/paulmck/scalability/paper/whymb.2010.06.07c.pdf)

2. [内存屏障Memory Barrier: a Hardware View](https://zhuanlan.zhihu.com/p/66085562)

3. [什么是内存屏障](https://zhuanlan.zhihu.com/p/602978542)

4. [Why Memory Barriers？中文翻译（上）](http://www.wowotech.net/kernel_synchronization/Why-Memory-Barriers.html)

5. [为什么需要内存屏障](https://zhuanlan.zhihu.com/p/55767485)

6. [你应该了解的 memory barrier 背后细节](https://mp.weixin.qq.com/s?__biz=Mzg3MzY1MzE1Nw==&mid=2247483852&idx=1&sn=bf06e008b5dbfee3020b741a834ab142&chksm=ceddf706f9aa7e101f974467e3c2f0aeaa7d03627fd3d9ba30ff0ff1d1956c8d91918d3246a0&mpshare=1&scene=1&srcid=0109bEiCOGxP7P3jgY5a9Ru0&sharer_shareinfo=725f66ac55220fb54f7f0e65e9b621f4&sharer_shareinfo_first=5080bb3ad85b3432128ade13a58fc815&version=4.1.16.99385&platform=mac#rd)

7. [CPU cache学习笔记](https://zhuanlan.zhihu.com/p/659708109)

8. [自底向上理解memory_order](https://zhuanlan.zhihu.com/p/682286231)
