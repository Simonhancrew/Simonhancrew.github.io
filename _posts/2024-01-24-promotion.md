---
layout: post
title:  "promotion"
category: c++
date:   2024-01-24
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

