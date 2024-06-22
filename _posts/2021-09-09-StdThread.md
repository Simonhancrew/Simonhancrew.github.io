---
title: 现代c++并发编程1-线程创建
date: 2021-09-09 14:10:00 +0800
categories: [Blogging]
tags: [writing]
---

c++语言层面没有提供进程级别的原语支持，所以你只用关注c++层面的线程并发

区别于并发和并行，前者是指多个任务交替执行，后者是指多个任务同时执行

因此，当某个场景说的并发的时候，你可能要理解一下他到底指什么

1. 多核机器，真的并行
2. 单个核心，任务切换

## 1. 线程的使用

在c++11中，线程完全依赖于`std::thread`类，这个类的构造函数接受一个可调用对象，这个对象会在新线程中执行

### 1.1 从hello world开始

```cpp
#include <iostream>
#include <thread>

int main() {
  std::thread t1{[]() { std::cout << "Hello from t1\n"; }};
  t1.join();
}
```

首先看到的是`std::thread`的构造函数，这个构造函数接受一个可调用对象，这个对象会在新线程中执行。

`t1.join()`是等待线程结束, 否则会一直阻塞在这里，直到线程结束。这里的调用时必须的，否则会导致thread析构的时候检查到joinable为true，调用std::terminate()终止程序。

因此不难知道，在join结束之后，thread绑定的线程就不再活跃，这个时候可以调用`joinable()`来检查线程是否还活跃。

### 1.2 线程相关的属性

`std::thread::hardware_concurrency();`, 这个函数会返回一个unsigned int，表示支持的并发数目，如果返回0，表示不支持多线程。

另外，`std::this_thread`包含4个函数，

1. `std::this_thread::get_id()`, 返回当前线程的id
2. `std::this_thread::sleep_for()`, 休眠一段时间
3. `std::this_thread::sleep_until()`, 休眠到某个时间点
4. `std::this_thread::yield()`, 让出cpu，建议实现重新调度各执行线程

### 1.3 线程的构造函数和结束

默认构造下，没有活跃线程会与这个t相关联。

其余情况下，只要传递给thread的是一个callable对象，那么这个对象会在新线程中执行。

callable大概有两种，一种是function类的对象，另一种就是实现了operator()的类对象。

但要注意的是，operator()的类对象，就没法直接想函数指针那样传递了。

```cpp
struct func {
  void operator()() { std::cout << "Hello from func\n"; }
};

int main() {
  std::thread t1{func{}};
  t1.join();
}
```

另外在这种情况下，还有一点值得注意，如果你使用`()`初始化了函数对象，那么这个其实会看作一个声名，比如`std::thread t(func());`。

启动线程后（也就是构造`std::thread`对象）我们必须在线程对象的生存期结束之前，即`std::thread::~thread`调用之前，决定它的执行策略，是`join()（合并)`还是`detach()（分离`。

因为先前简单介绍了join，这里主要讲一下detach，线程对象调用了 detach()，那么就是线程对象放弃了对线程资源的所有权，不再管理此线程，允许此线程独立的运行，在线程退出时释放所有分配的资源。

可以理解为此时`std::thread t`也没有活跃的线程与之相关联，放弃了对线程资源的所有权。

但这种写法其实值得注意，因为不会阻塞在detach，所以如果分离的线程还在访问父线程的资源，那么可能会导致未定义行为。

### 1.4 参数传递

正常情况下，thread的参数传递很简单，直接写在构造函数里就可以了。

但需要注意的是，这些参数会复制到新线程的内存空间中，即使函数中的参数是引用，依然实际是复制

```cpp
void f(int, const int& a);

int n = 1;
std::thread t{ f, 3, n };
```

如果想要传递引用，可以使用`std::ref`，这个函数会返回一个`std::reference_wrapper`对象，这个对象可以被拷贝，但是拷贝的是引用。

```cpp
void f(int, int& a) {
    std::cout << &a << '\n'; 
}

int main() {
    int n = 1;
    std::cout << &n << '\n';
    std::thread t { f, 3, std::ref(n) };
    t.join();
}
```

### 1.5 jthread

c++20的jthread相较于thread，可以看作只是多了两个功能

1. RAII，自动在析构时调用join
2. 线程停止

std::jthread 的通常实现就是单纯的保有 std::thread + [std::stop_source](https://zh.cppreference.com/w/cpp/thread/stop_source) 这两个数据成员：

关于线程停止，C++ 的 std::jthread 提供的线程停止功能并不同于常见的 POSIX 函数 pthread_cancel。pthread_cancel 是一种发送取消请求的函数，但并不是强制性的线程终止方式。目标线程的可取消性状态和类型决定了取消何时生效。当取消被执行时，进行清理和终止线程

std::jthread 所谓的线程停止只是一种基于用户代码的控制机制，而不是一种与操作系统系统有关系的线程终止。使用 std::stop_source 和 std::stop_token 提供了一种优雅地请求线程停止的方式，但实际上停止的决定和实现都由用户代码来完成。

```cpp
using namespace std::literals::chrono_literals;

void f(std::stop_token stop_token, int value){
    while (!stop_token.stop_requested()){ // 检查是否已经收到停止请求
        std::cout << value++ << ' ' << std::flush;
        std::this_thread::sleep_for(200ms);
    }
    std::cout << std::endl;
}

int main(){
    std::jthread thread{ f, 1 }; // 打印 1..15 大约 3 秒
    std::this_thread::sleep_for(3s);
    // jthread 的析构函数调用 request_stop() 和 join()。
}
```
