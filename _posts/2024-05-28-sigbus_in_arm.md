---
title: debug an arm SIGBUS
date: 2024-05-28 12:29:00 +0800
categories: [Blogging, c++, align]
tags: [writing]
---

arm下因为内存不对齐导致的SIGBUS问题

```
Revision: '0'
ABI: 'arm'
Happend: 'Sat May 25 02:07:23 2024'
SYSVMTYPE: Art
APPVMTYPE: Art
pid: 13178, tid: 13198, name: roidJUnitRunner
signal 7 (SIGBUS), code 1 (BUS_ADRALN), fault addr 0xdf652c09
    r0  00000000  r1  c2943ca8  r2  00000001  r3  dbecb083
    r4  00000000  r5  df652c09  r6  c2943ca8  r7  c1186c38
    r8  00000007  r9  00000039  r10 df652c09  r11 00000001
    ip  aca8e7f8  sp  c1186c10  lr  a3ae1b23  pc  a3ae1b2a
```

比较明显的发现0xdf652c09这个地址不是4/8的倍数，导致了SIGBUS的问题。但这个在arm64 + x86下都没有出现core。

代码里把8个uint8强转成了i64，导致了内存不对齐的问题。

+ 对于int32_t（32位整数），其地址应该是4的倍数，因为4字节（32位）是其自然对齐边界

+ 对于int64_t（64位整数），其地址应该是8的倍数，因为8字节（64位）是其自然对齐边界。

如果不满足这些对齐要求，某些处理器（如ARM）可能会抛出硬件异常（如SIGBUS），而其他处理器（如x86）可能会默默地进行更慢的非对齐访问。

```
std::vector<char> data;
....
auto* p = data.data();
int64_t* p64 = reinterpret_cast<int64_t*>(p);
```
### 1 内存的读取

程序中的数据类型的字节数大小，其实和操作系统是多少位没有关系，而是由编译器决定的。数据类型占用的字节数取决于编译时选择的编译器是64位还是32位。

为了保证每个对象拥有彼此独立的内存地址，C++空类的内存大小为1字节，而非空类的大小与类中非静态成员变量和虚函数表的多少有关。

其中，类中非静态成员变量的大小则与编译器的位数以及内存对齐的设置有关。类中的成员变量在内存中并不一定是连续的，它是按照编译器的设置，按照内存块(缓存行/cache-line)来存储的，这个内存块大小的取值，就是内存对齐。

这里比较关键的一点就是内存读取按照块来的，在有些平台上，如果遇到内存不对齐，就直接死给你看

### 2 为什么要内存对齐

+ 平台原因

不是所有的CPU都能访问任意地址上的任意数据的，有些CPU只能在某些地址处取某些特定类型的数据，否则抛出硬件异常。

+ 性能原因

数据结构应该尽可能地在自然边界上对齐，如果为了访问未对齐的内存，处理器需要作两次内存访问，而对齐的内存访问仅需要一次访问。CPU处理器把内存当作一块一块去读取，块的大小可以是2、4、8、16字节大小，这个大小称为内存存取粒度。假设当前处理器的内存存取粒度为4，对于一个int变量（大小4字节），分两种情况讨论：

1. 数据从0字节开始存放，CPU只需要访存一次，就可以把4字节数据读完，然后存入寄存器。
2. 数据从1字节开始存放，CPU要访问两次，才能把值写到寄存器。第一次先访问[0, 3]字节进入寄存器，第二次访问[4, 7]字节进入寄存器，然后剔除第0、5、6、7字节，仅留第1、2、3、4字节数据进入寄存器。对于未内存对齐的数据，显然大大降低了CPU的处理性能。这种未对齐的情况有些CPU甚至直接开摆。

### 3 内存对齐的规则

#### 3.1 #pragma对齐

用预编译命令`#pragma pack(n)`用来指定对齐系数，单位字节。n的取值范围为1, 2, 4, 8, 16。gcc默认对齐系数是4，msvc默认对齐系数是8。

用预编译命令`#pragma pack()`用来取消自定义的对齐系数，恢复为默认值。

```
//#pragma pack(4) //默认对齐系数是4
#pragma pack(1) //修改对齐系数为1

...

#pragma pack() //取消自定义的对齐系数，恢复为默认值4
```

假设在一个结构体中，最大数据类型长度为`m`，编译器对齐系数是`n`，则`min(m, n)`叫做对齐单位`s`。所以当设置的对齐系数`n`大于类中最大数据类型长度，该设置是不起作用的。当`n`等于1时，整个类的大小为所有成员长度之和。

所以可以有三个内存对齐规则

1. 每个成员的对齐规则：类中第一个成员的偏移量（`offset`）为0，以后每个成员（该成员的数据类型长度为`k`）相对于结构体首地址的offset为`min(k, s)`的整数倍。

2. 如果一个类里有结构体成员，则结构体成员要从其内部最宽基本类型成员的整数倍地址开始存储。

3. 整体对齐规则：整个结构体的大小应是对齐单位s的整数倍。

#### 3.2 c++里的对齐

c++11之后有`alignas`关键字来强制要求数据对齐, 每种对象类型都有一种名为对齐要求`alignment requirement`的属性，它是一个非负整数值, 类型为`std::size_t`，并且始终为2的幂，表示在这种类型的对象可以被分配的连续地址之间的字节数。看看cpp ref里的[alignas](https://en.cppreference.com/w/cpp/language/alignas)

```cpp
alignas(4) char buffer[sizeof(int) * (10 + 1)];
```

+ 类型的对齐要求可以使用[`alignof`](https://en.cppreference.com/w/cpp/language/alignof)或[`std::alignment_of`](https://en.cppreference.com/w/cpp/types/alignment_of)查询

+ 指针对齐函数`std::align`可用于在某个缓冲区中获取适当对齐的指针，`std::aligned_storage`可用于获取适当对齐的存储空间。

#### 3.3 c++里的空类

C++不允许一个对象的大小为0，不同对象的地址不能具有相同的地址。

这是因为`new`需要分配不同的内存地址，不能分配内存大小为0的空间，避免除以`sizeof(T)`引发除零异常。

所以一个没有数据成员的空类，编译器会为其分配1字节的内存空间，即空类的大小为1。

空基类被继承后，如果派生类有自己的数据成员，那么空基类这1个字节不会添加到派生类中

### 4 如果针对未对齐的内存做转义

一般来讲，在-O2下，不用vectorize的情况下，直接用memcpy是比较安全的方式，arm下也能work，但是可能性能会损失，如果O3下，编译器会尝试自动向量化，且出现了问题(编译器的bug)，记得在编译选项里加上`-fno-builtin-memcpy`。

### REF 

1. [Debugging a futex crash](https://rustylife.github.io/2023/08/15/futex-crash.html)
2. [位域](https://zhxilin.github.io/post/tech_stack/1_programming_language/modern_cpp/language_base/bit_field/)
3. [Memory alignment](https://docs.kernel.org/arch/arm/mem_alignment.html)
4. [内存对齐问题](https://blog.codingnow.com/2021/08/unalignment_memory_access.html)
5. [Take advantage of ARM unaligned memory access while writing clean C code](https://stackoverflow.com/questions/32062894/take-advantage-of-arm-unaligned-memory-access-while-writing-clean-c-code)
