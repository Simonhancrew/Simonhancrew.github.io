---
title: leveldb代码走读-no-destructor
date: 2024-07-25 11:54:00 +0800
categories: [Blogging, leveldb, no-destructor, code reading]
tags: [writing]
---

在leveldb里发现了一段代码

```cpp
// Wraps an instance whose destructor is never called.
//
// This is intended for use with function-level static variables.
template <typename InstanceType>
class NoDestructor {
 public:
  template <typename... ConstructorArgTypes>
  explicit NoDestructor(ConstructorArgTypes&&... constructor_args) {
    static_assert(sizeof(instance_storage_) >= sizeof(InstanceType),
                  "instance_storage_ is not large enough to hold the instance");
    static_assert(
        alignof(decltype(instance_storage_)) >= alignof(InstanceType),
        "instance_storage_ does not meet the instance's alignment requirement");
    new (&instance_storage_)
        InstanceType(std::forward<ConstructorArgTypes>(constructor_args)...);
  }

  ~NoDestructor() = default;

  NoDestructor(const NoDestructor&) = delete;
  NoDestructor& operator=(const NoDestructor&) = delete;

  InstanceType* get() {
    return reinterpret_cast<InstanceType*>(&instance_storage_);
  }

 private:
  typename std::aligned_storage<sizeof(InstanceType),
                                alignof(InstanceType)>::type instance_storage_;
};
```

简单看起来就是placement new之后的对象要自己call一次析构才能走到析构的过程

静态断言只是在检查空间是不是足够大，能够容纳要使用的InstanceType。

之前没怎么用过std::aligned_storage，用于创建一块具有特定大小和对齐要求的未初始化存储空间。它主要用于实现自定义内存管理和优化对象布局，特别是在需要手动控制对象生命周期的场景中。

好处是在不触发构造函数的情况下分配对象的内存，实现类型擦除和自定义内存布局，但c++23里弃用了，见REF2，用array也可以替代(17之后用std::byte数组)

leveldb里，这个一般配合单例使用，程序退出的时候，单例也会被销毁（析构），但是这玩意没有任何逻辑来析构instance_storage_。 所以其实里面storage保留的类对象其实永远不会被call析构。

## 为什么

c++并没有规定不同编译单元下的静态局部变量析构顺序，如果静态对象之前存在依赖关系，那么这个析构其实是有问题的

找到一篇参考的问题-->REF1


## REF 

1. [Safe Static Initialization, No Destruction](https://ppwwyyxx.com/blog/2023/Safe-Static-Initialization-No-Destruction/)
2. [Why is std::aligned_storage to be deprecated in C++23 and what to use instead?](https://stackoverflow.com/questions/71828288/why-is-stdaligned-storage-to-be-deprecated-in-c23-and-what-to-use-instead)
