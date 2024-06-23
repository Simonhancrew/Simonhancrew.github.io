---
title: 现代c++并发编程3-同步
date: 2024-06-23 14:10:00 +0800
categories: [Blogging, c++, thread, mutex, condtion_variable]
tags: [writing]
---

在多线程编程中，各个任务通常需要通过同步操作进行相互协调和等待，以确保数据的一致性和正确性

在c++中，你能想到的同步原语有：

1. 条件变量配合mutex
2. future/promise/packaged_task
3. c++20的latch/barrier

## 1. spin等待，不让出线程

```cpp
bool flag = false;
std::mutex m;

void wait_for_flag(){
    std::unique_lock<std::mutex> lk{ m };
    while (!flag){
        lk.unlock();    // 1 解锁互斥量
        lk.lock();      // 2 上锁互斥量
    }
}
```

[忙碌等待](https://zh.wikipedia.org/wiki/%E5%BF%99%E7%A2%8C%E7%AD%89%E5%BE%85), 一般来说，忙碌等待是应该避免的, 处理器时间应该用来执行其他任务，而不是浪费在无用的活动上。

## 2. 条件唤醒

C++ 标准库对条件变量有两套实现：[`std::condition_variable`](https://zh.cppreference.com/w/cpp/thread/condition_variable)和[`std::condition_variable_any`](https://zh.cppreference.com/w/cpp/thread/condition_variable_any)，这两个实现都包含在`<condition_variable>`头文件中。

any可以看作是condition_variable的泛化版本，可以和任何互斥量一起使用，只要能满足[基本锁定](https://zh.cppreference.com/w/cpp/named_req/BasicLockable)的要求, 但一般来说，我们使用condition_variable就够了。

```cpp
class Foo {
public:
  int tag_ = 0;
  std::condition_variable cv_;
  std::mutex mx_;

  Foo() {}

  void first(function<void()> printFirst) {
    // printFirst() outputs "first". Do not change or remove this line.
    printFirst();
    tag_ = 1;
    cv_.notify_all();
  }

  void second(function<void()> printSecond) {
    std::unique_lock<std::mutex> lk(mx_);
    cv_.wait(lk, [this]() { return tag_ == 1; });
    // printSecond() outputs "second". Do not change or remove this line.
    printSecond();
    tag_ = 2;
    lk.unlock();
    cv_.notify_one();
  }

  void third(function<void()> printThird) {
    std::unique_lock<std::mutex> lk(mx_);
    cv_.wait(lk, [this]() { return tag_ == 2; });
    // printThird() outputs "third". Do not change or remove this line.
    printThird();
    tag_ = 3;
    lk.unlock();
    cv_.notify_one();
  }
};
```

cv的wait主要考虑三件事

1. unique_lock锁住mutex
2. cv.wait(lock, predicate), 阻塞当前的线程，释放锁，知道条件满足(这个在gcc里就是简单的while循环)
3. 一旦条件满足，且cv被唤醒（包括虚假唤醒），当前线程重新获得锁，执行后续的操作

简单的说

```cpp
void wait(std::unique_lock<std::mutex>& lock);                 // 1

template<class Predicate>
void wait(std::unique_lock<std::mutex>& lock, Predicate pred); // 2
```

后者等价于

```cpp
while (!pred()) wait(lock);
```

这种其实比较方便解决[虚假唤醒](https://en.wikipedia.org/wiki/Spurious_wakeup)

## 3. 使用future

future在我的理解里，主要是为了方便解决thread之间的数据传递问题，比如一个线程计算一个值，另一个线程需要这个值，那么就可以使用future来传递这个值。

它用于处理线程中需要等待某个事件的情况，线程知道预期结果。等待的同时也可以执行其它的任务。

c++中有两种future，独占的`std::future` 、共享的`std::shared_future`。

类似于std::unique_ptr和std::shared_ptr的关系。

`std::future`只能与单个指定事件关联，而`std::shared_future`能关联多个事件。它们都是模板，它们的模板类型参数，就是其关联的事件（函数）的返回类型。当多个线程需要访问一个独立`future`对象时， 必须使用互斥量或类似同步机制进行保护。而多个线程访问同一共享状态，若每个线程都是通过其自身的`shared_future`对象副本进行访问，则是安全的。

### 3.1 配合std::async使用

相较于std::thread, std::aync提供了一个返回值的接口，可以方便的获取线程的返回值。

```cpp
#include <iostream>
#include <thread>
#include <future> // 引入 future 头文件

int task(int n) {
    std::cout << "异步任务 ID: " << std::this_thread::get_id() << '\n';
    return n * n;
}

int main() {
    std::future<int> future = std::async(task, 10);
    std::cout << "main: " << std::this_thread::get_id() << '\n';
    std::cout << std::boolalpha << future.valid() << '\n'; // true
    std::cout << future.get() << '\n';
    std::cout << std::boolalpha << future.valid() << '\n'; // false
}
```

使用`std::async`启动一个异步任务，它会返回一个`std::future`对象，这个对象和任务关联，将持有最终计算出来的结果。

当需要任务执行完的结果的时候，只需要调用`get()`成员函数，就会阻塞直到`future`为就绪为止（即任务执行完毕），返回执行结果。

`valid()`成员函数检查`future`当前是否关联共享状态，即是否当前关联任务。还未关联，或者任务已经执行完（调用了`get()`、`set()`），都会返回 false

async和thread的构造非常的类似，都支持任意的callable对象，包括支持std::ref，内部会将保有的参数副本转换为右值表达式进行传递

另外要关注的就是`std::async`的执行策略了，它能选择的策略只有两个

1. `std::launch::async` 在不同线程上执行异步任务。
2. `std::launch::deferred`惰性求值，不创建线程，等待future对象调用`wait`或`get`成员函数的时候执行任务。

他的[默认策略](https://github.com/microsoft/STL/blob/f54203f/stl/inc/future#L1425-L1430)，在你不给出显式的策略的时候，是`std::launch::async | std::launch::deferred`，即系统自己选择。

值得注意的是，在msvc的实现中，launch::async | launch::deferred 与 launch::async 执行策略毫无区别。

简而言之，使用`std::async`，只要不是`launch::deferred`策略，那么MSVC实现中都是必然在线程中执行任务。

且因为是线程池，所以执行新任务是否创建新线程，任务执行完毕线程是否立即销毁，不确定。

另外还有两点值得注意：

+ 如果从 std::async 获得的 std::future 没有被移动或绑定到引用，那么在完整表达式结尾，`std::future`的析构函数将阻塞到异步计算完成（阻塞在当前行）。因为临时对象的生存期就在这一行，调用析构函数阻塞执行。所以别这么用，这样写就不异步了

```cpp
std::async(std::launch::async, []{ f(); }); // 临时量的析构函数等待 f()
std::async(std::launch::async, []{ g(); }); // f() 完成前不开始
```

+ 被移动的`std::future`没有所有权，失去共享状态，不能调用 get、wait 成员函数。会抛异常的

### 3.2 配合std::packaged_task使用

`std::packaged_task`是一个可调用对象的包装器，它可以将任何可调用对象（函数、lambda表达式、函数对象）包装成一个`std::future`对象。单纯只是一个lambda的function

```cpp
std::packaged_task<double(int, int)> task([](int a, int b){
    return std::pow(a, b);
});
std::future<double>future = task.get_future();
task(10, 2); // 此处执行任务
std::cout << future.get() << '\n'; // 不堵塞，此处获取返回值
```

如果要在另一个线程执行这个task，并且使用future获取返回值，可以这样

```cpp
std::packaged_task<double(int, int)> task([](int a, int b){
    return std::pow(a, b);
});
std::future<double>future = task.get_future();
std::thread t{ std::move(task),10,2 }; // 任务在线程中执行
// todo.. 幻想还有许多耗时的代码
t.join();

std::cout << future.get() << '\n'; // 并不堵塞，获取任务返回值罢了
```

需要注意的是我们使用了 std::move ，这是因为 std::packaged_task 只能移动，不能复制

### 3.3 std::promise

类模板`std::promise`用于存储一个值或一个异常，之后通过`std::promise`对象所创建的`std::future`对象异步获得。

```cpp
// 计算函数，接受一个整数并返回它的平方
void calculate_square(std::promise<int> promiseObj, int num) {
    // 模拟一些计算
    std::this_thread::sleep_for(std::chrono::seconds(1));

    // 计算平方并设置值到 promise 中
    promiseObj.set_value(num * num);
}

// 创建一个 promise 对象，用于存储计算结果
std::promise<int> promise;

// 从 promise 获取 future 对象进行关联
std::future<int> future = promise.get_future();

// 启动一个线程进行计算
int num = 5;
std::thread t(calculate_square, std::move(promise), num);

// 阻塞，直到结果可用
int result = future.get();
std::cout << num << " 的平方是：" << result << std::endl;

t.join();
```

我们在新线程中通过调用`set_value()`函数设置promise的值，并在主线程中通过与其关联的future对象的 `get()` 成员函数获取这个值，如果promise的值还没有被设置，那么将阻塞当前线程，直到被设置为止。

同样的`std::promise`只能移动，不可复制，所以我们使用了`std::move`进行传递。

除了`set_value()`函数外，`std::promise`还有一个`set_exception()`成员函数，它接受一个`std::exception_ptr`类型的参数，这个参数通常通过`std::current_exception()`获取，用于指示当前线程中抛出的异常。然后，`std::future`对象通过`get()`函数获取这个异常，如果promise所在的函数有异常被抛出，则`std::future`对象会重新抛出这个异常，从而允许主线程捕获并处理它。

```cpp
void throw_function(std::promise<int> prom) {
    try {
        throw std::runtime_error("一个异常");
    }
    catch (...) {
        prom.set_exception(std::current_exception());
    }
}

int main() {
    std::promise<int> prom;
    std::future<int> fut = prom.get_future();

    std::thread t(throw_function, std::move(prom));

    try {
        std::cout << "等待线程执行，抛出异常并设置\n";
        fut.get();
    }
    catch (std::exception& e) {
        std::cerr << "来自线程的异常: " << e.what() << '\n';
    }
    t.join();
}
```

另外，promise只能储存一个值或异常，值和异常无法共存。

简而言之，`set_value`与`set_exception`二选一，如果先前调用了`set_value`，就不可再次调用`set_exception`

## 4. std::shared_future + std::future的状态

future 是一次性的，所以你需要注意移动。并且，调用get函数后，future对象也会失去共享状态。

如果需要进行多次`get`调用，考虑使用`std::shared_future`。

### 4.1 future的局限

很多线程在等待的时候，只有一个线程能获取结果。当多个线程等待相同事件的结果时，就需要使用std::shared_future来多次get结果了

需要关注的是，在多个线程中对同一个std::shared_future对象进行操作时（如果没有进行同步保护）存在竞争条件。

而从多个线程访问同一共享状态，若每个线程都是通过其自身的shared_future对象副本进行访问，则是安全的。

```cpp
std::string fetch_data() {
    std::this_thread::sleep_for(std::chrono::seconds(1)); // 模拟耗时操作
    return "从网络获取的数据！";
}

int main() {
    std::future<std::string> future_data = std::async(std::launch::async, fetch_data);

    // // 转移共享状态，原来的 future 被清空  valid() == false
    std::shared_future<std::string> shared_future_data = future_data.share();

    // 第一个线程等待结果并访问数据
    std::thread thread1([shared_future_data] {
        std::cout << "线程1：等待数据中..." << std::endl;
        shared_future_data.wait();
        std::cout << "线程1：收到数据：" << shared_future_data.get() << std::endl;
    });

    // 第二个线程等待结果并访问数据
    std::thread thread2([shared_future_data] {
        std::cout << "线程2：等待数据中..." << std::endl;
        shared_future_data.wait();
        std::cout << "线程2：收到数据：" << shared_future_data.get() << std::endl;
    });

    thread1.join();
    thread2.join();
}
```

注意这里是按照copy传递给lambda的，如果是一个引用，那么就会出现同一个shared_future而导致竞争条件。

在c++20中，还存在`<semaphore>`和`<latch> + <barrier>`这种玩意，看起来感觉有点像内存屏障了，跟cv + mutex虽然都能做线程同步，但是不准备放在一起写

## REF

1. [同步操作](https://github.com/Mq-b/ModernCpp-ConcurrentProgramming-Tutorial/blob/main/md/04%E5%90%8C%E6%AD%A5%E6%93%8D%E4%BD%9C.md)
