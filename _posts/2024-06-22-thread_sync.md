---
title: 现代c++并发编程2-mutex
date: 2024-06-22 14:10:00 +0800
categories: [Blogging, c++, thread, mutex, condtion_variable]
tags: [writing]
---

线程安全相关的同步原语，主要解决表达式冲突问题，即多个线程同时访问共享数据，导致数据不一致的问题。

注意有三种情况是例外的

1. 同线程处理
2. atomic操作
3. memory barrier下的内存序已经能确认的操作

## 1. mutex

mutex是一种用来保护临界区的同步原语，其相当于实现了一个公共的“标志位”。它可以处于锁定（locked）状态，也可以处于解锁（unlocked）状态：

1. 如果互斥量是锁定的，通常说某个特定的线程正持有这个锁。

2. 如果没有线程持有这个互斥量，那么这个互斥量就处于解锁状态。

多次运行下面的代码，输出基本没有规律

> c++里同步的流是线程安全的

```cpp
void f() {
    std::cout << std::this_thread::get_id() << '\n';
}

int main() {
    std::vector<std::thread> threads;
    for (std::size_t i = 0; i < 10; ++i)
        threads.emplace_back(f);

    for (auto& thread : threads)
        thread.join();
}
```

因此需要一个标志来保护线程的执行顺序

```cpp
#include <mutex> // 必要标头
std::mutex m;

void f() {
    m.lock();
    std::cout << std::this_thread::get_id() << '\n';
    m.unlock();
}

int main() {
    std::vector<std::thread>threads;
    for (std::size_t i = 0; i < 10; ++i)
        threads.emplace_back(f);

    for (auto& thread : threads)
        thread.join();
}
```

当多个线程执行函数 f 的时候，只有一个线程能成功调用 lock() 给互斥量上锁，其他所有的线程 lock() 的调用将阻塞执行，直至获得锁。第一个调用 lock() 的线程得以继续往下执行，执行我们的 std::cout 输出语句，不会有任何其他的线程打断这个操作。直到线程执行 unlock()，就解锁了互斥量。

那么其他线程此时也就能再有一个成功调用`lock`

> 至于到底哪个线程才会成功调用，这个是由操作系统调度决定的。

### 1.2 std::lock_guard

一般推荐使用lock_guard去做互斥量的保护，因为lock_guard是RAII的，不需要手动unlock

```cpp
void f() {
    std::lock_guard<std::mutex>lc{ m };
    std::cout << std::this_thread::get_id() << '\n';
}
```

同时它还提供一个有额外std::adopt_lock_t参数的构造函数 ，如果使用这个构造函数，则构造函数不会上锁。

以有的时候你可能会看到一些这样的代码：

```cpp
void f(){
    //code..
    {
        std::lock_guard<std::mutex> lc{ m };
        // 涉及共享资源的修改的代码...
    }
    //code..
}
```

并且 C++17 还引入了一个新的“管理类”：std::scoped_lock，它相较于 lock_guard的区别在于，它可以管理多个互斥量。不过对于处理一个互斥量的情况，它和 lock_guard 几乎完全相同。

### 1.3 try_lock

try_lock 是互斥量中的一种尝试上锁的方式。与常规的 lock 不同，try_lock 会尝试上锁，但如果锁已经被其他线程占用，则不会阻塞当前线程，而是立即返回。

它的返回类型是 bool ，如果上锁成功就返回 true，失败就返回 false。

这种方法在多线程编程中很有用，特别是在需要保护临界区的同时，又不想线程因为等待锁而阻塞的情况下。

```cpp
std::mutex mtx;

void thread_function(int id) {
    // 尝试加锁
    if (mtx.try_lock()) {
        std::cout << "线程：" << id << " 获得锁" << std::endl;
        // 临界区代码
        std::this_thread::sleep_for(std::chrono::milliseconds(100)); // 模拟临界区操作
        mtx.unlock(); // 解锁
        std::cout << "线程：" << id << " 释放锁" << std::endl;
    } else {
        std::cout << "线程：" << id << " 获取锁失败 处理步骤" << std::endl;
    }
}
```

### 1.4 死锁问题

两个线程需要对它们所有的互斥量做一些操作，其中每个线程都有一个互斥量，且等待另一个线程的互斥量解锁。因为它们都在等待对方释放互斥量，没有线程工作。 这种情况就是死锁。

避免死锁的一般建议是让两个互斥量以相同的顺序上锁，总在互斥量 B 之前锁住互斥量 A，就通常不会死锁

C++ 标准库有很多办法解决这个问题，可以使用 std::lock ，它能一次性锁住多个互斥量，并且没有死锁风险

```cpp
void swap(X& lhs, X& rhs) {
    if (&lhs == &rhs) return;
    std::lock(lhs.m, rhs.m);    // 给两个互斥量上锁
    std::lock_guard<std::mutex> lock1{ lhs.m,std::adopt_lock }; 
    std::lock_guard<std::mutex> lock2{ rhs.m,std::adopt_lock }; 
    swap(lhs.object, rhs.object);
}
```

因为前面已经使用了 std::lock 上锁，所以后的 std::lock_guard 构造都额外传递了一个 std::adopt_lock 参数，让其选择到不上锁的构造函数。函数退出也能正常解锁。

当然，如果你是c++17的话，可以使用`std::scoped_lock`，它可以管理多个互斥量，不过对于处理一个互斥量的情况，它和 lock_guard 几乎完全相同

```cpp
void swap(X& lhs, X& rhs) {
    if (&lhs == &rhs) return;
    std::scoped_lock guard{ lhs.m,rhs.m };
    swap(lhs.object, rhs.object);
}
```

### 1.5 std::unique_lock, 支持unlock

std::unique_lock 是 C++11 引入的一种通用互斥包装器，它相比于 std::lock_guard 更加的灵活。当然，它也更加的复杂，很多情况下，他一般与 std::condition_variable 一起使用。

```cpp
void swap(X& lhs, X& rhs) {
    if (&lhs == &rhs) return;
    std::unique_lock<std::mutex> lock1{ lhs.m, std::defer_lock };
    std::unique_lock<std::mutex> lock2{ rhs.m, std::defer_lock };
    std::lock(lock1, lock2);
    swap(lhs.object, rhs.object);
    ++n;
}
```

std::defer_lock 是“不获得互斥体的所有权”。没有所有权自然构造函数就不会上锁

可以简单看下unique_lock的实现成员

```cpp
private:
    _Mutex* _Pmtx = nullptr;
    bool _Owns    = false;

unique_lock(_Mutex& _Mtx, defer_lock_t) noexcept
    : _Pmtx(_STD addressof(_Mtx)), _Owns(false) {} // construct but don't lock

```

并且 std::unique_lock 是有 lock() 、try_lock() 、unlock() 成员函数的，所以可以直接传递给`std::lock`进行调用。

```cpp
void lock() { // lock the mutex
    _Validate();
    _Pmtx->lock();
    _Owns = true;
}
```

但比较gay的是，有一种情况下，unique_lock是有mutex的所有权的且没有上锁的，这种情况下，直接call lock是会抛异常的

```cpp
void _Validate() const { // check if the mutex can be locked
    if (!_Pmtx) {
        _Throw_system_error(errc::operation_not_permitted);
    }

    if (_Owns) {
        _Throw_system_error(errc::resource_deadlock_would_occur);
    }
}
```

所以你其实要针对mutex额外的进行锁,也就是说 std::unique_lock 要想调用 lock() 成员函数，必须是当前没有所有权。

```cpp
lock.mutex()->lock();
```

因此可以简单的总结一下

1. 使用 std::defer_lock 构造函数不上锁，要求构造之后上锁
2. 使用 std::adopt_lock 构造函数不上锁，要求在构造之前互斥量上锁
3. 默认构造会上锁，要求构造函数之前和构造函数之后都不能再次上锁

## 2. 互斥量的不同作用域传递

互斥量本身是不可复制 + 不可移动的，但是他的指针 + 引用可以。

所以可以用各种lock类来做传递

std::unique_lock 可以获取互斥量的所有权，而互斥量的所有权可以通过移动操作转移给其他的 std::unique_lock 对象。有些时候，这种转移（就是调用移动构造）是自动发生的，比如当函数返回 std::unique_lock 对象。另一种情况就是得显式使用 std::move。

std::unique_lock 是只能移动不可复制的类，它移动即标志其管理的互斥量的所有权转移了。

```cpp
std::unique_lock<std::mutex>get_lock(){
    extern std::mutex some_mutex;
    std::unique_lock<std::mutex> lk{ some_mutex };
    return lk;

}
void process_data(){
    std::unique_lock<std::mutex> lk{ get_lock() };
    // 执行一些任务...
}
```

要注意，`extern std::mutex some_mutex`，如果你简单写一个`std::mutex some_mutex`那么函数`process_data`中的 lk 会持有一个悬垂指针。

## 3. 共享变量的初始化

保护共享数据并非必须使用互斥量，互斥量只是其中一种常见的方式而已，对于一些特殊的场景，也有专门的保护方式，比如对于共享数据的初始化过程的保护。我们通常就不会用互斥量，这会造成很多的额外开销。

这里一般有三种方式，正确保证初始化

1. 使用`std::call_once`
2. 局部static变量初始化

首先看第一个

```cpp
std::shared_ptr<some>ptr;
std::mutex m;
std::once_flag resource_flag;

void init_resource(){
    ptr.reset(new some);
}

void foo(){
    std::call_once(resource_flag, init_resource); // 线程安全的一次初始化
    ptr->do_something();
}
```

以上代码 std::once_flag 对象是全局命名空间作用域声明，如果你有需要，它也可以是类的成员。用于搭配 std::call_once 使用，保证线程安全的一次初始化。std::call_once 只需要接受可调用 (Callable)对象即可，也不要求一定是函数。

当然如果你考虑到异常的话，std::call_once 也有一些例外情况（比如异常）会让传入的可调用对象被多次调用。

```cpp
std::once_flag flag;
int n = 0;

void f(){
    std::call_once(flag, [] {
        ++n;
        std::cout << "第" << n << "次调用\n";
        throw std::runtime_error("异常");
    });
}

int main(){
    try{
        f();
    }
    catch (std::exception&){}
    
    try{
        f();
    }
    catch (std::exception&){}
}
```

另外一种就是c++11之后的单例

```cpp
class my_class;
inline my_class& get_my_class_instance(){
    static my_class instance;
    return instance;
}
```

## 4. 读写操作方面的锁

有些读多写少的情况，mutex其实比较重，所以在c++17的时候提供了`std::shared_mutex`, c++14有`std::shared_timed_mutex`

`std::shared_mutex` 同样支持 `std::lock_guard`、`std::unique_lock`。和 s`td::mutex` 做的一样，保证写线程的独占访问。而那些无需修改数据结构的读线程，可以使用 `std::shared_lock<std::shared_mutex>` 获取访问权，多个线程可以一起读取。

```cpp
class Settings {
private:
    std::map<std::string, std::string> data_;
    mutable std::shared_mutex mutex_; // “M&M 规则”：mutable 与 mutex 一起出现

public:
    void set(const std::string& key, const std::string& value) {
        std::lock_guard<std::shared_mutex> lock{ mutex_ };
        data_[key] = value;
    }

    std::string get(const std::string& key) const {
        std::shared_lock<std::shared_mutex> lock(mutex_);
        auto it = data_.find(key);
        return (it != data_.end()) ? it->second : ""; // 如果没有找到键返回空字符串
    }
};
```

std::shared_timed_mutex 具有 std::shared_mutex 的所有功能，并且额外支持超时功能。所以以上代码可以随意更换这两个互斥量

## 5. 递归锁

线程对已经上锁的 std::mutex 再次上锁是错误的，这是未定义行为。然而在某些情况下，一个线程会尝试在释放一个互斥量前多次获取，所以提供了`std::recursive_mutex`。

`std::recursive_mutex`是 C++ 标准库提供的一种互斥量类型，它允许同一线程多次锁定同一个互斥量，而不会造成死锁。当同一线程多次对同一个`std::recursive_mutex`进行锁定时，只有在解锁与锁定次数相匹配时，互斥量才会真正释放。但它并不影响不同线程对同一个互斥量进行锁定的情况。不同线程对同一个互斥量进行锁定时，会按照互斥量的规则进行阻塞，

```cpp
#include <iostream>
#include <thread>
#include <mutex>

std::recursive_mutex mtx;

void recursive_function(int count) {
    // 递归函数，每次递归都会锁定互斥量
    mtx.lock();
    std::cout << "Locked by thread: " << std::this_thread::get_id() << ", count: " << count << std::endl;
    if (count > 0) {
        recursive_function(count - 1); // 递归调用
    }
    mtx.unlock(); // 解锁互斥量
}

int main() {
    std::thread t1(recursive_function, 3);
    std::thread t2(recursive_function, 2);

    t1.join();
    t2.join();
}
```

同样的，我们也可以使用 std::lock_guard、std::unique_lock 帮我们管理 std::recursive_mutex，而非显式调用 lock 与 unlock

## 6. TLS

线程存储期（也有人喜欢称作“线程局部存储”）的概念源自操作系统，是一种非常古老的机制，广泛应用于各种编程语言。线程存储期的对象在线程开始时分配，并在线程结束时释放。每个线程拥有自己独立的对象实例，互不干扰。在 C++11中，引入了`thread_local`关键字，用于声明具有线程存储期的对象。

```cpp
int global_counter = 0;
thread_local int thread_local_counter = 0;

void print_counters(){
    std::cout << "global：" << global_counter++ << '\n';
    std::cout << "thread_local：" << thread_local_counter++ << '\n';
}

int main(){
    std::thread{ print_counters }.join();
    std::thread{ print_counters }.join();
}
```

每一个线程都有独立的 thread_local_counter 对象，它们不是同一个。

