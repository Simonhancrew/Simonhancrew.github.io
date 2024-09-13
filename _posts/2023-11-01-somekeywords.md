---
title: inline + extern + static
date: 2023-11-01 14:10:00 +0800
categories: [Blogging, c++]
tags: [writing]
---

### inline

很早就知道std=c++11的inline其实和之前不一样了，11之后的编译器，几乎不会参考函数前的inline建议

另外inline其实不仅仅可以作用在函数头上，其实也可以作用在namespace上，在17之后甚至可以修饰变量

在11之后，inline很关键的一个作用时允许同一个函数或者是变量的定义出现在多个编译单元中，而不会报错, 最重要的其实也是针对这种ODR行为

其实我目前觉得就inline函数比较有用

#### inline函数

inline func主要修饰一个external linkage

若一个非static的函数，在被多个编译单元重复定义，那么这个函数就必须是inline的，否则会报错。这样编译器就会认为当前的符号是一个弱符号，面对多个的时候之后保留一个定义

当然，下面的也可以用static解决，这时候函数不会被export，只会用自己编译单元的函数，不会报错。


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

对于一些很小的函数，直接在头文件里写inline的定义，这种可能会因为编译器在编译的时候在#include h file的时候能看到完整的定义，会更容易的自行去inline，这里的inline就是真的展开了

但现代编译器比你想象要聪明很多，写在.cpp里这样写，可能他也会在编译完这个translation unit之后在使用这个函数的地方自动再inline

讨论这些其实意义不大

## static

storage class specifier

static的作用也分

1. 函数内的局部变量，只有在第一次掉用的时候才初始化，但注意，这个变量的生命周期是整个程序的生命周期，而不是函数的生命周期，而且不一定是线程安全的(c++ 11 singleton对于多线程的支持，不要new)

2. class内的静态成员变量，所有的对象共享，但是必须在类外初始化(当然针对一些类型可以内联完成，内联的staic const就经常被拿做类内部的常量用), 但是这个变量的生命周期是整个程序的生命周期，而不是类的生命周期。

3. 在namespace里，static的作用是限制变量的作用域，只能在当前的编译单元中使用，再在外部用extern再倒入的时候，就会发生链接错误

4. 当然上面说的几个，基本不是在cpp里定义static，就是在class内部，或者是在namespace内部，这里的static都是internal linkage的，也就是说在其他的编译单元中是不可见的，这里的static就是internal linkage的。但是你也可以傻逼，在头文件里定义static，这样就会导致每个cpp文件中都有一个static的副本，这个副本是local的，可能会导致程序中有大量的static副本。贼几把大

然后类内部的static成员变量，要用的时候，记得在cpp里定义或者声明(const static定义了的那种，可以直接在类内部定义，但是在cpp里要取地址的话，还是要声明一下，比如你要用min/max之类的), 这种属于ODR use的范畴，只要不要求地址的，其实大概率都可以额编译+链接过。如果工程不打，编译优化级别开的够高，这里也可能被优化掉，在[An In-depth Look at C++ Keyword: static](https://mp.weixin.qq.com/s?__biz=MzUxOTQ4NjIzNw==&mid=2247488215&idx=1&sn=4aa9df0a34e701f37ab0234f489f2db9&chksm=f9f9bca5ce8e35b3500fe7995b31ec5b5b925187760c6e3cd17a9008b96e85b1f7aefe6c0812&cur_album_id=2980489080029511688&scene=189#wechat_redirect)里有详细的描写

有个[回答](https://stackoverflow.com/questions/3698043/in-header-files-what-is-the-difference-between-a-static-global-variable-and-a-s)比较好，可以看看.

之前提到的odr use，static和inline，还有声明 + 实现分离都能处理，可以看一下这三者在符号可见性的区别

|          | test.o                   | test1.o                   |
|----------|--------------------------|---------------------------|
| origin error | 00000000000 T fun       | 00000000000000 T fun      |
| 定义实现分离   | U fun                    | U fun                     |
| static       | 00000000000 t fun       | 00000000000000 t fun      |
| inline       | 00000000000 W fun       | 00000000000000 W fun      |


符号可见性方面

| 字符 | 描述                                                                                     |
|------|------------------------------------------------------------------------------------------|
| T    | 这个符号表示一个全局或静态函数，其定义在代码段中。也就是说，符号 T 表示这是一个在目标文件中定义的可执行代码部分 |
| U    | 这个符号表示一个未定义的外部符号。也就是说，目标文件中引用了这个符号，但它的定义于其他文件或目标文件中，通常在链接阶段解析 |
| t    | 这个符号表示一个局部符号，在代码段中定义。也就是说，符号 t 表示这是一个仅在目标文件内部有有效的局部函数（或局部变量），不会被其他文件引用 |
| W    | 这个符号表示一个弱符号。弱符号在链接时如果有其他强符号与之定义，弱符号会被忽略。弱符号通常用于提供默认实现，但允许其他文件中进行覆盖 |



## extern

extern告诉编译器某个声明对于其他源文件中的代码是可见的。也就是说，声明具有链接性质。

这意味着所有具有该名称的实体都指的是声明为extern的实体。

因此，如果你想在多个文件中使用一个全局变量，应该使用extern去声明该变量：extern int global_i.该声明可以出现在头文件或源文件中。

如果将其放入源文件，则必须在使用global_i的每个源文件中重复声明。

最后，全局是有问题的，跨越源文件的全局更是如此。请小心使用extern。

## linkage

具有external linkage的变量可以被其它源文件使用，整个程序内有效，并且全局只有一个

具有internal linkage的变量只能被本translation unit所引用，如果有多个translation unit，则会有多个副本, 每个cpp文件中都会有一个

一般来讲，全局的non-const变量会具有external linkage，但是加上static之后就会具有internal linkage

1. 编译单元
  
    一个.cpp文件或者.c文件才称为一个translation unit，头文件(.h)不能称为一个translation unit，因为其最终会被加(plus)入到include它们的.cpp/.c文件中去，编译器编译的是.cpp/.c文件，而不是.h文件。

2. 缺省的链接方式
    
      在缺省情况下，常量(const)具有static属性(internal linkage)，而非常量(non-const)具有extern属性(external linkage)

      常量(const)因为具有static的internal linkage属性，其会导致每个使用它的cpp文件中都会包含一个const副本，这个副本是local的，可能会导致程序中有大量的常量副本。

      变量具有extern的external linkage属性，如果在头文件中定义变量并且该头文件被多个cpp文件引用，将导致重复定义的编译错误；对于变量正确的做法应该是在cpp文件中定义，然后在使用它的cpp文件中使用extern来声明它，即不要将变量直接暴露在.h文件中，如果一定要在头文件中暴露变量，那么应使用extern来修饰它。


### 匿名namespace

匿名namespace天然具有internal linkage的属性，因此在头文件中使用匿名namespace来定义常量是一个不错的选择，因为它不会导致重复定义的编译错误，也不会导致大量的常量副本。

ps，虽然namespace默认是external linkage的，但匿名namespace默认情况下会给每一个cpp文件中的匿名namesapce生成一个唯一的name，这个在别的空间里没法拿到而已， 所以在gg code style里面也会大规模的推荐使用匿名namespace来代替static，可以少写很多static

在匿名命名空间中声明的名称也将被编译器转换，与编译器为这个匿名命名空间生成的唯一内部名称(即这里的_UNIQUE_NAME_)绑定在一起。

## Ref

1. [internal-linkage](https://www.learncpp.com/cpp-tutorial/internal-linkage/)

2. [Scope, duration, and linkage summary](https://www.learncpp.com/cpp-tutorial/scope-duration-and-linkage-summary/)

3. [External linkage and variable forward declarations](https://www.learncpp.com/cpp-tutorial/external-linkage-and-variable-forward-declarations/)

4. [An In-depth Look at C++ Keyword: static](https://mp.weixin.qq.com/s?__biz=MzUxOTQ4NjIzNw==&mid=2247488215&idx=1&sn=4aa9df0a34e701f37ab0234f489f2db9&chksm=f9f9bca5ce8e35b3500fe7995b31ec5b5b925187760c6e3cd17a9008b96e85b1f7aefe6c0812&cur_album_id=2980489080029511688&scene=189#wechat_redirect)
