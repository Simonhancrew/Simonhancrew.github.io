---
title: 继承顺序之类的问题
date: 2025-01-21 10:20:00 +0800
categories: [Blogging, c++]
tags: [writing]
---

### 基础继承

先看一个不包含虚函数的例子：

```cpp
#include <iostream>

using namespace std;

class A {
public:
  A() { cout << "Construct A" << endl; }
  ~A() { cout << "Destruct A" << endl; }
};

class C {
public:
  C() { cout << "Construct C" << endl; }
  ~C() { cout << "Destruct C" << endl; }
};

class B : public A, public C {
public:
  B() { cout << "Construct B" << endl; }
  ~B() { cout << "Destruct B" << endl; }
};

int main() {
  B b;
  return 0;
}
```

这个的输出结果是

```bash
Construct A
Construct C
Construct B
Destruct B
Destruct C
Destruct A
```

这里构造的顺序是先父类然后子类，父类的顺序看起来是从左到右。这里不包含初始列表还有虚继承之类的。

我试了一下，如果把B强行写成下面的样子，输出没什么变化

```cpp
class B : public A, public C {
public:
  B() : C(), A() { cout << "Construct B" << endl; }
  ~B() { cout << "Destruct B" << endl; }
};
```

### 基础组合

如果变一下，B变成组合A + C, 这里看到成员顺序是先C再A，初始化列表是先A再C的，理论上我要的顺序是先A再C

```cpp
class B {
public:
  B() : a(A()), c(C()) { cout << "Construct B" << endl; }
  ~B() { cout << "Destruct B" << endl; }
  C c;
  A a;
};
```

输出结果是

```bash
Construct C
Construct A
Construct B
Destruct B
Destruct A
Destruct C
```

看起来列表初始化的顺序和数据成员定义顺序相关，跟摆放位置其实是没关系的


### 带上虚继承

虚继承（virtual inheritance）是一种C++中的继承方式，用于解决多重继承中可能出现的菱形继承问题。菱形继承问题是指在多重继承中，派生类通过多个路径继承同一个基类，导致基类的成员在派生类中出现多次。

通过使用虚继承，基类在派生类中只会有一个实例，从而避免了重复继承的问题。

比如这样一个例子

```cpp
#include <iostream>
using namespace std;

class A {
public:
  A() { cout << "Construct A" << endl; }
  ~A() { cout << "Destruct A" << endl; }
};

class B : virtual public A {
public:
  B() { cout << "Construct B" << endl; }
  ~B() { cout << "Destruct B" << endl; }
};

class C : virtual public A {
public:
  C() { cout << "Construct C" << endl; }
  ~C() { cout << "Destruct C" << endl; }
};

class D : public B, public C {
public:
  D() { cout << "Construct D" << endl; }
  ~D() { cout << "Destruct D" << endl; }
};

int main() {
  D d;
  return 0;
}
```

这里的结果输出

```bash
Construct A
Construct B
Construct C
Construct D
Destruct D
Destruct C
Destruct B
Destruct A
```

如果不用virtual继承的话，那么在B构造和C构造的时候还是会继续A的构造

过渡到我们要看的例子，看下构造顺序

```cpp
class B : public A, public virtual C {
public:
  B() : a(A()), c(C()) { cout << "Construct B" << endl; }
  ~B() { cout << "Destruct B" << endl; }
  C c;
  A a;
};
```

输出结果是

```bash
Construct C
Construct A
Construct C
Construct A
Construct B
Destruct B
Destruct A
Destruct C
Destruct A
Destruct C
```

先执行虚拟继承的父类的构造函数，然后从左到右执行普通继承的父类的构造函数，然后按照定义的顺序执行数据成员的初始化，最后是自身的构造函数的调用。

### 总结

在类被构造的时候，先执行虚拟继承的父类的构造函数，然后从左到右执行普通继承的父类的构造函数，然后按照定义的顺序执行数据成员的初始化，最后是自身的构造函数的调用。析构函数与之完全相反，互成镜像。
