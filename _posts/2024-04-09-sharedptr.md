---
layout: post
title:  "shared ptr"
category: c++
date:   2024-04-09
---

shared_ptr内部主要有两个点，第一个是contrl block，第二个是内部储存的类空间

ctrl block是在堆上创建的，内部会有一个atomic变量来做引用计数的增减, 因此，shared_ptr的copy其实是线程安全的，这也意味着修改指向内容的操作不是线程安全的。

这里谈到第二个点，为什么推荐使用make_shared，原因之一是RAII问题，new构造 + shared_ptr构造可能不在同一时期，中间抛一场的话可能导致内存泄漏

```cpp
vector<shared_ptr<A>> rec;
rec.emplace_back(new A);
```

第二个原因就是llvm的实现下，new一个shared ptr看起来会new两次，分别ctrl + T, 但是make shared看起来会直接要一片大内存，本质上其实只new了一次

### 一些要注意的点

1. 尽量用make_shared
2. shared_ptr可以自定义deleter，而且比unique_ptr好的点是可以直接写成lambda
3. 不用拿裸指针创建多个shared_ptr   
4. 类返回this->shared_ptr的话，记得enale_shared_from_this + shared_from_this, 因为他本质也是个指针，跟3一样


### REF

1. [shared_ptr线程安全](https://www.zhihu.com/question/56836059)

