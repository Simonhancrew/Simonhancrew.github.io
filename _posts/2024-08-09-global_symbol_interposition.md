---
title: 全局符号介入
date: 2024-08-09 21:10:00 +0800
categories: [Blogging, compile, link]
tags: [writing]
---

全局符号介入是一种机制，允许一个符号（通常是一个函数）被另一个同名符号替换或覆盖。这通常发生在动态链接过程中。

```
/*
# ALLOCATED CHUNKS

# +---------------------------+
# |         METADATA          |
# |---------------------------|  <-- Pointer returned by malloc points to
# |                           |      the user data section of the chunk
# |         USER DATA         |
# |                           |
# |---------------------------|
# |         METADATA          |
# +---------------------------+  } Chunk 1
# +---------------------------+
# |         METADATA          |
# |---------------------------|
# |                           |
# |         USER DATA         |
# |                           |
# |---------------------------|
# |         METADATA          |
# +---------------------------+  } Chunk 2
*/
```


假设有这样一个场景

1. A.so和B.so都有一个header.h的文件，各自include这个文件到cpp中，这个文件里有一个info结构体，实现了info.reset()方法
2. B.so中，使用new Data()创建一个data对象，这个时候其实分两步走了
   1. sizeof(data), 这个行为是编译时，所以这个内部info的size是在编译时确定的
   2. new (mem) Data()， 里面会调用info.reset()，这个行为是在运行时，所以这个info.reset()是在运行时确定的
3. 此时，执行这个构造的时候，先找到了A.so中的info.reset()，然后执行了A.so中的info.reset()，后面的B.so中的info.reset()都会被忽略

因此，他用了自己的info结构体，执行了别人的构造函数，这就是全局符号介入的问题。如果

新结构体的reset函数里额外的成员赋值导致了写入内存超过了老结构体的size，堆内存越界

## 怎么调查

ldd查看链接顺序，具体的符号查看nm， 找到相同的符号，这个符号一般是W的，弱符号。

## REF

+ [轻踩一下就崩溃吗——踩内存案例分析](https://mp.weixin.qq.com/s/9OCFb2cH-H5zbaIT5VAS9w)
