---
title: event loop实现 + 事件添加
date: 2024-07-23 01:08:00 +0800
categories: [Blogging, event loop]
tags: [writing]
---

看了下业界的eventloop实现，大多数的思路都是one thread per loop，然后在fd增加之类的，大多数都是在回调事件里，会跟wait同时进行

比较骚的是，我看大家打断epoll_wait都用的是event fd read来触发

我在想这个是不是在某些场景下其实会影响并发的性能。。。

我浅看了一下muduo的实现是这样的，但workflow的实现看起来又是另外一套。

workflow是一个线程在往epoll里+，另外一个线程在epoll wait，这样的好处是可能性能上占了点优势，但是我不确定这样写是不是好兼容poll之类的。后面兼容win下的iocp会好扩展嘛，poller感觉会有点难写。

## 1. muduo event loop

TBD

## REF

1. [workflow QA](https://github.com/sogou/workflow/issues/170)

