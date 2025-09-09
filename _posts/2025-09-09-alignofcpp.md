---
title: c++里的对齐语义
date: 2025-09-09 11:18:00 +0800
categories: [Blogging, cpp, align]
tags: [writing]
---

大多数结构都要求地址按照align对齐，这个align值一般是4或者8。c++中的一些在align场景下常用的关键字

### 1 alignment

常规意义下，对齐意味着当前变量的起始地址是针对align做mod的结果是0的，具体的看代码

```cpp
#include <iostream>
#include <cstdint>

using namespace std;

#define IS_ALIGN_OF(ptr, alignment)                                            \
  do {                                                                         \
    size_t val = (reinterpret_cast<uintptr_t>(ptr) % alignment);               \
    if (val != 0) {                                                            \
      cout << "Address " << ptr << " is not aligned to " << alignment << endl; \
    }                                                                          \
  } while (0)

#define IS_ALIGN_OF_COMPILE_TIME(ptr, alignment)                               \
  do {                                                                         \
    static_assert(alignof(decltype(*ptr)) == alignment, "alignment mismatch"); \
  } while (0)

int main() {
  int a;
  char b;
  short c;
  long d;
  float e;
  double f;
  long long g;
  long double h;
  cout << "a: " << &a << endl;
  cout << "b: " << static_cast<void*>(&b) << '\n';
  cout << "c: " << &c << endl;
  cout << "d: " << &d << endl;
  cout << "e: " << &e << endl;
  cout << "f: " << &f << endl;
  cout << "g: " << &g << endl;
  cout << "h: " << &h << endl;

  cout << "a: " << sizeof(a) << endl;
  cout << "b: " << sizeof(b) << endl;
  cout << "c: " << sizeof(c) << endl;
  cout << "d: " << sizeof(d) << endl;
  cout << "e: " << sizeof(e) << endl;
  cout << "f: " << sizeof(f) << endl;
  cout << "g: " << sizeof(g) << endl;
  cout << "h: " << sizeof(h) << endl;

  IS_ALIGN_OF(&a, sizeof(a));
  IS_ALIGN_OF_COMPILE_TIME(&a, sizeof(a));
  IS_ALIGN_OF(&b, sizeof(b));
  IS_ALIGN_OF_COMPILE_TIME(&b, sizeof(b));
  IS_ALIGN_OF(&c, sizeof(c));
  IS_ALIGN_OF_COMPILE_TIME(&c, sizeof(c));
  IS_ALIGN_OF(&d, sizeof(d));
  IS_ALIGN_OF(&e, sizeof(e));
  IS_ALIGN_OF_COMPILE_TIME(&e, sizeof(e));
  IS_ALIGN_OF(&f, sizeof(f));
  IS_ALIGN_OF_COMPILE_TIME(&f, sizeof(f));
  IS_ALIGN_OF(&g, sizeof(g));
  IS_ALIGN_OF_COMPILE_TIME(&g, sizeof(g));
  IS_ALIGN_OF(&h, sizeof(h));
  IS_ALIGN_OF_COMPILE_TIME(&h, sizeof(h));

  // IS_ALIGN_OF_COMPILE_TIME(&b, sizeof(h));

  return 0;
}
```

### 2 alignof

c++11的语言内置的元信息查询机制，能看到T的类型对齐要求

```cpp
#include <cstdint>
#include <iostream>
#include <type_traits>

using namespace std;

int main() {
  int a;
  char b;
  short c;
  long d;
  float e;
  double f;
  long long g;
  long double h;
  std::cout << "alignof(a): " << alignof(decltype(a)) << '\n';
  std::cout << "alignof(b): " << alignof(decltype(b)) << '\n';
  std::cout << "alignof(c): " << alignof(decltype(c)) << '\n';
  std::cout << "alignof(d): " << alignof(decltype(d)) << '\n';
  std::cout << "alignof(e): " << alignof(decltype(e)) << '\n';
  std::cout << "alignof(f): " << alignof(decltype(f)) << '\n';
  std::cout << "alignof(g): " << alignof(decltype(g)) << '\n';
  std::cout << "alignof(h): " << alignof(decltype(h)) << '\n';
  return 0;
}
```

更清楚的看下结构体size align, 这里总体的sizeof是24，结构体的align是8，struct这种数据类型的offset可以用gnu的macro offset来看结构体成员到结构体初始地址的偏移

```cpp
#include <cstdint>
#include <iostream>
#include <type_traits>

using namespace std;

struct Test {
  char a;
  int32_t b;
  int32_t c;
  uint64_t d;
};

// #define offsetof(OBJECT_TYPE, MEMBER) __builtin_offsetof(OBJECT_TYPE, MEMBER)

int main() {
  std::cout << "size of Test: " << sizeof(Test) << '\n';
  std::cout << "alignof Test: " << alignof(Test) << '\n';
  std::cout << "offsetof a: " << offsetof(Test, a) << '\n';
  std::cout << "offsetof b: " << offsetof(Test, b) << '\n';
  std::cout << "offsetof c: " << offsetof(Test, c) << '\n';
  std::cout << "offsetof d: " << offsetof(Test, d) << '\n';
  return 0;
}
```

这里的输出是

```
size of Test: 24
alignof Test: 8
offsetof a: 0
offsetof b: 4
offsetof c: 8
offsetof d: 16
```

回到最开始说的内存对齐，每个指针的地址要满足mod align的值是0

因此这里b的偏移4，中间要填充3才能满足对齐要求， b跟c之前没有偏移，c到d之间有有4字节的偏移，要填充4才能满足8字节的对齐要求

因此这个结构的安排类似

```ascii
+----+----+----+----+----+----+----+----+
| 0  | 1  | 2  | 3  | 4  | 5  | 6  | 7  |
+----+----+----+----+----+----+----+----+
| a  |    |    |    | b  | b  | b  | b  |
+----+----+----+----+----+----+----+----+
| 8  | 9  | 10 | 11 | 12 | 13 | 14 | 15 |
+----+----+----+----+----+----+----+----+
| c  | c  | c  | c  |    |    |    |    |
+----+----+----+----+----+----+----+----+
| 16 | 17 | 18 | 19 | 20 | 21 | 22 | 23 |
+----+----+----+----+----+----+----+----+
| d  | d  | d  | d  | d  | d  | d  | d  |
+----+----+----+----+----+----+----+----+
```

alignof到结构体，只要最大的字段满足对齐，别的字段也是满足对齐的

### 3 alignas

需要改变一个结构的align细节，c++可以借助alignas关键字

比如之前的Test结构体，继续针对他做alignas

```cpp
struct alignas(16) Test {
  char a;
  int32_t b;
  int32_t c;
  uint64_t d;
};
```

输出的值是

```zsh
size of Test: 32
alignof Test: 16
offsetof a: 0
offsetof b: 4
offsetof c: 8
offsetof d: 16
```

等价内存layout是

```ascii
+----+----+----+----+----+----+----+----+
| 0  | 1  | 2  | 3  | 4  | 5  | 6  | 7  |
+----+----+----+----+----+----+----+----+
| a  |    |    |    | b  | b  | b  | b  |
+----+----+----+----+----+----+----+----+
| 8  | 9  | 10 | 11 | 12 | 13 | 14 | 15 |
+----+----+----+----+----+----+----+----+
| c  | c  | c  | c  |    |    |    |    |
+----+----+----+----+----+----+----+----+
| 16 | 17 | 18 | 19 | 20 | 21 | 22 | 23 |
+----+----+----+----+----+----+----+----+
| d  | d  | d  | d  | d  | d  | d  | d  |
+----+----+----+----+----+----+----+----+
| 24 | 25 | 26 | 27 | 28 | 29 | 30 | 31 |
+----+----+----+----+----+----+----+----+
|    |    |    |    |    |    |    |    |
+----+----+----+----+----+----+----+----+
```

但这个是有限制的

1. 不能小于align(T)的值
2. 大概率得是2的幂，不然也会报错(Requested alignment is not a power of 2clang(alignment_not_power_of_two))

### 4 std::aligned_storage

```cpp
typedef __type_list<__align_type<unsigned char>,
        __type_list<__align_type<unsigned short>,
        __type_list<__align_type<unsigned int>,
        __type_list<__align_type<unsigned long>,
        __type_list<__align_type<unsigned long long>,
        __type_list<__align_type<double>,
        __type_list<__align_type<long double>,
        __type_list<__align_type<__struct_double>,
        __type_list<__align_type<__struct_double4>,
        __type_list<__align_type<int*>,
        __nat
        > > > > > > > > > > __all_types;

template <class _Hp, class _Tp, size_t _Len>
struct __find_max_align<__type_list<_Hp, _Tp>, _Len>
    : public integral_constant<size_t, __select_align<_Len, _Hp::value, __find_max_align<_Tp, _Len>::value>::value> {};


template <size_t _Len, size_t _Align = __find_max_align<__all_types, _Len>::value>
struct _LIBCPP_DEPRECATED_IN_CXX23 _LIBCPP_TEMPLATE_VIS aligned_storage {
  typedef typename __find_pod<__all_types, _Align>::type _Aligner;
  union type {
    _Aligner __align;
    unsigned char __data[(_Len + _Align - 1) / _Align * _Align];
  };
};
```

这里是c++11引入的，分配长度len，满足align为align的内存(默认是当前系统最大对齐)

这里保证了分配大小是align内存的倍数，对len / align进行上取整，最后aligner是all type::type，这里保证了是结构体是按照_align进行对齐的

### 5 std::align

```cpp
/// @param  __align 内存对齐align值
/// @param  __sz 想要内存大小
/// @param  __ptr 输入指向ptr地址，输出时调整为符合alignment对齐要求的内存地址
/// @param  __space 剩余待分配空间，输出的时候会被调整为剩下的空间(当前返回指针地址开始)
/// @return 如果 ptr 经过调整后能满足大小为 alignment 的对齐要求，则返回ptr的值，否则返回 nullptr
void* align(size_t __align, size_t __sz, void*& __ptr, size_t& __space);
```

从ptr取出一片满足对齐要求的地址

因此这个使用方案可以看看rocksdb里的arena

```cpp
template <size_t N>
struct Arena {
  char buffer[N];
  void* ptr;
  size_t size;

  Arena() : ptr(buffer), size(N) { }

  template <typename T>
  T* AlignedAllocate(size_t alignment = alignof(T)) {
      if (std::align(alignment, sizeof(T), ptr, size)) {
          T* result = reinterpret_cast<T*>(ptr);
          ptr = (char*)ptr + sizeof(T);
          size -= sizeof(T);
          return result;
      }
      return nullptr;
  }
};
```

一个可能的algn的实现，从gnu的libc++里弄出来的

```cpp
inline void* align(size_t __align, size_t __size, void *&__ptr, size_t &__space) noexcept {
    const auto __intptr = reinterpret_cast<uintptr_t>(__ptr);
    const auto __aligned = (__intptr - 1u + __align) & -__align;
    const auto __diff = __aligned - __intptr;
    if ((__size + __diff) > __space) {
        return nullptr;
    }
    __space -= __diff;
    return __ptr = reinterpret_cast<void *>(__aligned);
}
```

__aligned等价于上取证之后，把最高__align位之前的1值全部变成0，等价在找__align倍数

### 6 #pragma pack

默认情况下，编译器会根据平台、类型自动对齐（比如 int 对齐到 4 字节，double 到 8 字节），但有时为了节省空间或兼容硬件/协议（如网络包、文件格式），我们需要强制调整对齐方式

具体用的方法

```cpp
#pragma pack(push, N)      // 保存当前对齐状态，设置新对齐为 N 字节
// 结构体定义
#pragma pack(pop)  

// 默认12， align = 4
struct A {
    char a;    // 1 byte
    int b;     // 4 bytes → 要求 4-byte 对齐，补 3 字节空隙
    short c;   // 2 bytes
};
[ a ][___][___][___][b][b][b][b][c][c]
 0    1    2    3    4    5    6    7    8    9

#pragma pack(1)
struct B {
    char a;
    int b;
    short c;
};
#pragma pack()

[a][b][b][b][b][c][c]
 0  1  2  3  4  5  6
```

配合c++的algnof看看

```cpp
#include <cstdint>
#include <iostream>
#include <type_traits>

using namespace std;

// #define offsetof(OBJECT_TYPE, MEMBER) __builtin_offsetof(OBJECT_TYPE, MEMBER)
#pragma pack(1)
struct A {
  char a;
  int b;
  short c;
};
#pragma pack()

int main() {
  cout << "alignof A: " << alignof(A) << endl;
  return 0;
}
```

输出是1

这样用主要可能出现的问题

1. cacheline操作性能下降
2. arm的严格不容忍非对齐，会抛一个sig啥的，之前写过.
