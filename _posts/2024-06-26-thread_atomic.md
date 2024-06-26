---
title: 现代c++并发编程5-原子操作和内存序
date: 2024-06-26 14:10:00 +0800
categories: [Blogging, c++, thread, atomic, memory_order]
tags: [writing]
---

atomic在C++中其实被分成两个部分，第一个是原子操作，第二个是内存序屏障。

1. 原子操作，是指一个操作要么全部执行，要么全部不执行，不会被中断。原子操作是一种线程安全的操作，不需要额外的锁来保护。
2. 内存序屏障，是指在多线程并发编程中，为了保证内存序列不击穿，需要在一些地方插入内存序屏障，来保证内存序列的一致性。

如果要完全理解内存屏障，必然要从cache line说起

为了避免频繁的主存交互，其实缓存体系采用了类似malloc的方法，即划分一个最小单元，叫做缓存行（主流CPU上一般64B），所有内存到缓存的操作，以缓存行为单位整块完成。

例如对于连续访问来说第一个B的访问就会触发全部64B数据都进入L1，后续的63B访问就可以直接由L1提供服务了。所以并发访问中的第一个问题就是要考虑缓存行隔离，也就是一般可以认为，位于不同的两个缓存行的数据，是可以被真正独立加载/淘汰和转移的（因为cache间流转的最小单位是一个cache line）。

这一块有一个比较出名的现象, false sharing, 也就是不慎将两个本无竞争的数据，放置在一个缓存行内，导致因为体系结构的原因，引入了并不应该存在的竞争

在排除了false sharing的情况下，我们还需要考虑内存序的问题，也就是确实需要位于同一个缓存行内的数据（往往就是同一个数据），多个核心都要修改的场景。由于在多核心系统中cache存在多份，因此就需要考虑这多个副本间一致性的问题。

这个一致性一般由一套状态机协议保证（MESI及其变体）

大体是，当竞争写入发生时，需要竞争所有权，未获得所有权的核心，只能等待同步到修改的最新结果之后，才能继续自己的修改。

这里要提一下的是有个流传甚广的说法是，因为缓存系统的引入，带来了不一致，所以引发了各种多线程可见性问题。

这么说其实有失偏颇，MESI本质上是一个『一致性』协议，也就是遵守协议的缓存系统，其实对上层CPU多个核心做到了顺序一致性。比如对比一下就能发现，缓存在竞争时表现出来的处理动作，其实和只有主存时是一致的。



## REF

1. [浅谈Memory Reordering](http://dreamrunner.org/blog/2014/06/28/qian-tan-memory-reordering/)
2. [Is Parallel Programming Hard, And, If So, What Can You Do About It?](https://github.com/paulmckrcu/perfbook)
3. [perfbook](https://cdn.kernel.org/pub/linux/kernel/people/paulmck/perfbook/perfbook.html)
4. [百度C++工程师的那些极限优化（内存篇）](https://mp.weixin.qq.com/s/wF4M2pqlVq7KljaHAruRug)
