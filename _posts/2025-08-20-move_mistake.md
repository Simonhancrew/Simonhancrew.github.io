---
title: move使用的问题点
date: 2025-08-20 18:08:14 +0800
categories: [Blogging, move, c++, bug]
tags: [writing]
---

主要鉴赏一些move写出来的bug

### for里使用move

在for里使用move需要考虑细致一点，鉴定一个场景

```cpp
std::unique_ptr<Task> task(...)
for (0..theadpool.size()) {
    threadpool[i].run(std::move(task));
}

ThreadPool::run(UniqueTask task) {
    // without check task
    // ...
    task->run();
    // ...
}
```

这里是一个全网故障点QAQ
