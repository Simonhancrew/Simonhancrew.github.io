---
title: promotion
date: 2024-01-24 14:10:00 +0800
categories: [Blogging, c++]
tags: [writing]
---

c++里的一些类型promotion，先随便看看一元`+`操作符的使用技巧


不过若是操作数为整数或无作用域枚举类型，一元 + 操作符会执行 Integral promotion，此时会发生隐式转换。例如

```c++
// unscoped enumeration
enum Enum : unsigned int {
  enum_val_a,
  enum_val_b,
  enum_val_c
};

int main() {

  bool b = true;
  +b; // int

  +enum_val_b; // unsigned int

  char c = 'c';
  +c; // int

  unsigned short s = 10;
  +s; // int

  int array[10];
  +array; // int*
}
```

对于一些奇怪的类型，比如 std::uint8_t，它的类型是什么呢？顾名思义应该是 8-bit 的 Unsigned integer，然而实际上它是 unsigned char 的 typedef。那么在输出的时候就会遇到一些问题：


```c++
std::uint8_t u = 0x45;
std::cout << u; // E
```

这里的输出结果是 E，而不是 69，因为 std::cout 会将 char 类型的参数当作字符输出。而借助一元 + 操作符，则可以非常简单地达到预期。

```c++
std::uint8_t u = 0x45;
std::cout << +u; // 69
```

还有一个是lambda的问题，正常lambda的类型是一个closure，但是+之后可以转换成函数指针

```c++
auto fp = +[]{};
static_assert(std::is_same_v<decltype(fp), void (*)()>);
```

还有一些用法在concept中，不熟。

## char和uint8

char在大多情况下是个有符号数，有些编译器可能是无符号的。uint8是一个name alia

char转成uint8在[0, 127]是对的,[-128, -1]对应[128, 255]

在做字节流的时候互相用用可能还好，但是如果生成uint8, 传出去给int，这个时候再去拿int当无符号数就会发生傻逼事情。。。

所以在用无符号数的时候，尽量都用无符号数接着，不要转成有符号数再转回来

## 取模运算

准确的说这个不算promotion，只不过最近遇到了写一下。a % min(c, d);

要确保一件事，c和d里不要有0，否则会发生除0错误

除法和取模都要保证不要拿0做底
