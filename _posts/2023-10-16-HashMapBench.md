---
layout: post
title:  "HashMap Bench"
category: c++
date:   2023-10-16
---

标准库里的map和unordered_map，从某种程度上说，性能其实不是很行

## list

首先说list，不支持随机访问，每个节点都是分散的，在大多数情况下每一个遍历的步骤都是cache miss

当前热点如果真的不怎么访问这个list，随便用，如果热点访问list，找死了

## map and hashmap

map里面是一个红黑树，组织起来的就是list，插入和搜索其实都是partila search tree

当然不谈树旋转之类的操作，对内存层面来说，这些操作依然贵的一批。

现代的cpu足够快了，我们要的是更快的内存，这个可以看cppcon2014的演讲，[ppt在这里](https://github.com/CppCon/CppCon2014/blob/master/Presentations/Efficiency%20with%20Algorithms%2C%20Performance%20with%20Data%20Structures/Efficiency%20with%20Algorithms%2C%20Performance%20with%20Data%20Structures%20-%20Chandler%20Carruth%20-%20CppCon%202014.pdf)

hashmap里其实可以用数组做寻址，但是比比较不幸的是每个bucket的链接还是list。。。


## flat hash map and concrrent hash map

abseil里的实现[Abseil Containers](https://abseil.io/docs/cpp/guides/container)

flat hash map的作者还有个介绍[线程安全的map的博客](https://greg7mdp.github.io/parallel-hashmap/)


## REF

1. [flat_map性能调研](https://zhuanlan.zhihu.com/p/661418250)
2. [Comprehensive C++ Hashmap Benchmarks 2022](https://martin.ankerl.com/2022/08/27/hashmap-bench-01/)
3. [map_benchmark by marinus](https://github.com/martinus/map_benchmark)
4. [The Parallel Hashmap](https://greg7mdp.github.io/parallel-hashmap/)
5. [开源库 parallel-hashmap 介绍：高性能 线程安全 内存友好的哈希表 和 btree](https://byronhe.com/post/2020/11/10/parallel-hashmap-btree-fast-multi-thread-intro/)

