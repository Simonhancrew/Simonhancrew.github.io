---
title: 代码刷日志情况总结
date: 2024-08-14 21:10:00 +0800
categories: [Blogging, system programming, design]
tags: [writing]
---

目前碰到了好几种可能在代码里导致一直循环刷日志的情况，简单总结下。

目前看下来

1. 尽量不要把observer callback里的方法写的太复杂。
2. 检查timer的销毁时机，尽量用one shot timer
3. 不要在构造函数里做太多事情，就构造内存就好，真要做事情，用init函数

### 1. 0ms的repeat timer

第一种就不要例子了，0ms的repeat timer，不会取消，这种一直消耗loop的

### 2. callback再次回调自己

这种的根因还是设计问题，B和A互相耦合了，理论上讲，B是A的low level，B不应该感知到A。

但现实是，B确实有很多和A的交互，需要从b里call A的observer，这种就需要observer里的设计不要太复杂，尤其是不要再回到B。

```cpp
Foo a;
Bar B(a);

void Foo::callB() {
  // do something
  B.callback();
}

void Foo::callBImpl() {
  // do something
  callB();
} 

void Bar::callback() {
  // do something
  a.callBImpl();
}
```

这种其实有一个简单的想法可以处理, 但母鸡好不好用，还要看具体的业务逻辑。

```cpp

class ObserverBase {
private:
  bool inCallback = false;

protected:
  virtual void doCallback() = 0;

public:
  void observerCallback() {
    if (inCallback) return;
    inCallback = true;
    doCallback();
    inCallback = false;
  }
};

class ConcreteObserver : public ObserverBase {
protected:
  void doCallback() override {
      // 实际的回调逻辑
      lowerLayer.doSomething();
  }
};

Object::Notify() {
  observer.observerCallback();
}

```

### 3. 类构造不完全 + 结构耦合

这种构造就初始化内存，不要做复杂的操作。不要在构造里搞一堆调用的逻辑

```cpp
class A {
  std::unique_ptr<B> b;
  A() {
    b.reset(new B(this));
  }
  void CreateBSuccess() {
    auto b = getB();
  }

  auto getB() {
    if (!b) {
      b.reset(new B(this));
    }
    return b.get();
  }
};

class B {
  A* a;
  B(A* a) {
    this->a = a;
    a->CreateBSuccess();
  }
};
```
