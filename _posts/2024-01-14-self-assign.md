---
layout: post
title:  "self assign"
category: c++
date:   2024-01-14
---

大多数时候自赋值的检查是没有必要的，尤其是没有使用什么裸指针类型之类的。如果你自己的类里面全部是高级的类型，没有必要自己重载operator=

但有一种情况，一个类持有一个指针，且掌控这个指针的生命周期，这时候就需要自己重载operator=，来避免自赋值的情况。

```c++
class T {
int* p;

public:
  T(const T &rhs) : p(rhs.p ? new int(*rhs.p) : nullptr) {}
  ~T() { delete p; }

  // ...

  T& operator=(const T &rhs) {
    delete p;
    p = new int(*rhs.p);
    return *this;
  }
};
```

上述的行为有两种办法可以处理，第一种就是自赋值检查

```c++
class T {
int* p;

public:
  T(const T &rhs) : p(rhs.p ? new int(*rhs.p) : nullptr) {}
  ~T() { delete p; }

  // ...

  T& operator=(const T &rhs) {
    if(this == &rhs)
      return *this;

    delete p;
    p = new int(*rhs.p);
    return *this;
  }
};
```

第二种就是copy-swap

```c++
class T {
int* p;

public:
  T(const T &rhs) : p(rhs.p ? new int(*rhs.p) : nullptr) {}
  ~T() { delete p; }

  // ...

  void swap(T &rhs) {
    using std::swap;
    swap(p, rhs.p);
  }

  T& operator=(const T &rhs) {
    T(rhs).swap(*this);
    return *this;
  }
};
```

还有一种就是产生一个临时变量，然后move过去，这种看起来不是很优雅...


### REF

1. [bugprone-unhandled-self-assignment](https://clang.llvm.org/extra/clang-tidy/checks/bugprone/unhandled-self-assignment.html)

2. [OOP54-CPP. Gracefully handle self-copy assignment](https://wiki.sei.cmu.edu/confluence/display/cplusplus/OOP54-CPP.+Gracefully+handle+self-copy+assignment)

3. [What is wrong with "checking for self-assignment" and what does it mean?](https://stackoverflow.com/questions/12015156/what-is-wrong-with-checking-for-self-assignment-and-what-does-it-mean)
