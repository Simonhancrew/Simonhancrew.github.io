---
layout: post
title: "Lambda的capture变量的声明周期"
date: 2023-03-10
---

### Lambda的生命周期

正常来讲， lambda的所capture的值如果是引用的话， 需要注意一下生命周期的问题。

保证lambda不要获得比这些引用捕获的值长。

在debug一次crash的时候， 发现了一个和lambda生命周期有关的bug

```
[weak_ptr_, cb](int, vector<string>) {
  if (cb)  {
    cb(); // 这个cb可能会设置这个lambda为nullptr
  }
  weak_ptr_.lock();
  ...
}
```
这个lambda是被保存在一个对象的function中的， 在cb执行中，可能导致这个lambda的function被置为nullptr。 比较trick的是， 这个function被置为nullptr之后，*this没有意义了， 开始析构， 当前lambda所捕获的变量就会被销毁， 所以后续执行到的跟捕获有关的代码， 都是heap use after free的。



