---
layout: post
title:  "system optimize"
category: system-design
date:   2024-04-30
---

计算、存储、网络、硬件，主要是在这四个维度

首先要有一个足够准的钟，因此个机房配几个带GPS和/或原子钟primary standard的NTP服务器。

但是应该在同一个局域网的服务器之间还是存在一定的时钟差，所以要想办法做补偿(时间戳补偿，time sync， 延迟补偿)

另外网络上的延迟可以分成两大类，第一类是物理层面规则的限制，第二个是惯性延迟。

第一类很好理解，传输速度，跟距离有关

第二类跟硬件限制有关，跟网络带宽成反比，千兆网TCP有效带宽按115MB/s估算，那么发送1150字节的消息从第1个字节离开本机网卡到第1150个字节离开本机网卡至少需要 10us，这是无法降低的，因此必要的话可以减小消息长度。

找到瓶颈，如果在CPU的话，我觉得除了算法层面的优化，不如考虑怎么用缓存(哪些可以提前做？哪些可以不做？哪些可以做了然后存一段时间？)

在代码逻辑的小点上，大概能想到的

1. 限制内存分配，主要是动态的
2. 使用轮询，避免阻塞，尽量避免上下文切换
3. 使用共享内存作为IPC的唯一方式（共享内存只有在初始化的时候有一些系统调用，之后就可以像访问正常内存一样使用了。但这个可能要找个好点的实现，因为为了数据一致要做点东西），但能用单线程实现的就用单线程。。。
4. 无锁结构，找一个好点的消息队列实现
5. 尽量使用缓存友好的代码，绑核心（cpu affinity），别用list这种傻逼了
6. 让编译器做更多的事，放点东西到编译期，能用CRTP的就别用虚函数，尤其是hot path
7. 感觉最大的瓶颈还是在网络，机房位置选好，然后考虑下openload之类的东西
8. 异常问题，可以用点unsafe的代码，比如vector[] VS vector.at()，但是要保证不会出现越界
9. 用好编译器提供的builtins，像是__expected，__prefetch之类。
10. 注意好struct padding。也要留意在多线程情况下会出现的false sharing情况
11. 了解下代码优化级别，我记得有时候不一定是优化flag大的更好
  
另外收集了点资料，youtube上有几个视频，感觉还不错

1. [Carl Cook “When a Microsecond Is an Eternity: High Performance Trading Systems in C++”](https://www.youtube.com/watch?v=NH1Tta7purM&ab_channel=CppCon)
2. [Performance Tuning](https://www.youtube.com/watch?v=fV6qYho-XVs&ab_channel=MattGodbolt)

关于写出cache frienly的代码有点文章

1. [Memory part 5: What programmers can do](https://lwn.net/Articles/255364/)
2. [what is cache friendly](https://stackoverflow.com/questions/16699247/what-is-a-cache-friendly-code)
3. [Why does the order of the loops affect performance when iterating over a 2D array?](https://stackoverflow.com/questions/9936132/why-does-the-order-of-the-loops-affect-performance-when-iterating-over-a-2d-arra)
4. [Software optimization resources](https://www.agner.org/optimize/)
5. [Cache-Friendly-Code](https://merikanto.io/2018/Cache-Friendly-Code/)

最后一篇好像总结的还可以
