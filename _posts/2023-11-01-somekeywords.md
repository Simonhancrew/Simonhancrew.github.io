---
layout: post
title:  "inline + extern + static"
category: c++
date:   2023-11-01
---

### inline

很早就知道std=c++11的inline其实和之前不一样了，11之后的编译器，几乎不会参考函数前的inline建议

另外inline其实不仅仅可以作用在函数头上，其实也可以作用在namespace上，在17之后甚至可以修饰变量

在11之后，inline很关键的一个作用时允许同一个函数或者是变量的定义出现在多个编译单元中，而不会报错

其实我目前觉得就inline函数比较有用

#### inline函数

若一个非static的函数，在被多个编译单元重复定义，那么这个函数就必须是inline的，否则会报错。这样编译器就会认为当前的符号是一个弱符号，面对多个的时候之后保留一个定义


```cpp
/* foo.h */
inline int foo(int x) {
  static int factor = 1;
  return x * (factor++);
}

/* bar1.cc */
#include "foo.h"
int bar1() { return foo(1); }

/* bar2.cc */
#include "foo.h"
int bar2() { return foo(2); }

/* main.cc */
int Bar1(), Bar2();
int main() { return Bar1() + Bar2(); }
```

比较搞心态的是如果有多个这样的弱符号，那么编译器会保留一个，但是不会保证是哪一个，所以这样的代码你没法保证最后掉用的是那个具体的定义，就比如下面这种

```cpp
/* bar1.cc */
inline int Foo(int x) {
  static int factor = 1;
  return x * (factor++);
}
int Bar1() { return Foo(1); }

/* bar2.cc */
inline int Foo(int x) {
  static int factor = 2;
  return x * (factor++);
}
int Bar2() { return Foo(2); }

/* main.cc */
int Bar1(), Bar2();
int main() { return Bar1() + Bar2(); }
```

实际的结果可能是按照编译的顺序去做掉用的，所以我建议工程大了最好少在头文件用inline定义

## static

static的作用也分

1. 函数内的局部变量，只有在第一次掉用的时候才初始化，但注意，这个变量的生命周期是整个程序的生命周期，而不是函数的生命周期，而且不一定是线程安全的(c++ 11 singleton对于多线程的支持，不要new)

2. class内的静态成员变量，所有的对象共享，但是必须在类外初始化(当然针对一些类型可以内联完成，内联的staic const就经常被拿做类内部的常量用)

3. 在namespace里，static的作用是限制变量的作用域，只能在当前的编译单元中使用，再在外部用extern再倒入的时候，就会发生链接错误

## extern

extern告诉编译器某个声明对于其他源文件中的代码是可见的。也就是说，声明具有链接性质。

这意味着所有具有该名称的实体都指的是声明为extern的实体。

因此，如果你想在多个文件中使用一个全局变量，应该使用extern去声明该变量：extern int global_i.该声明可以出现在头文件或源文件中。

如果将其放入源文件，则必须在使用global_i的每个源文件中重复声明。

最后，全局是有问题的，跨越源文件的全局更是如此。请小心使用extern。
