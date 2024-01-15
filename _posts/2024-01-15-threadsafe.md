---
layout: post
title:  "线程安全"
category: os
date:   2024-01-15
---

线程到底会share哪些资源？

今天看到`inet_ntoa`这种函数，我记得在glibc下，稍微新一点的，都应该是线程安全的，但是在openbsd下面发现并不一定。之前游双那本书里写的放到现在不全对了。

Drepper大佬在[这个commit](https://github.com/bminor/glibc/commit/279d494b1d089d085b44f2de6c62a0752001b074)里修了这个问题。

但是apple能找到的部分，看起来还是没有改过...


### REF

1. [线程间到底共享了哪些进程资源？](https://cloud.tencent.com/developer/article/1768025)

2. [glibc-inet_ntoa](https://github.com/bminor/glibc/blob/master/inet/inet_ntoa.c)

3. [Thread-Local Storage](https://gcc.gnu.org/onlinedocs/gcc/Thread-Local.html)

4. [线程局部存储tls的使用](https://tboox.org/cn/2016/09/28/thread-local/)

5. [Thread-local storage wiki](https://en.wikipedia.org/wiki/Thread-local_storage)
