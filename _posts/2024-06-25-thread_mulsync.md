---
title: 现代c++并发编程4-多发同步
date: 2024-06-25 14:10:00 +0800
categories: [Blogging, c++, thread, latch, barrier, semaphore]
tags: [writing]
---

c++20里还有一些比较轻量的同步原语

1. semaphore
2. latch + barrier

## 1. semaphore, 信号量

[信号量](https://zh.wikipedia.org/wiki/%E4%BF%A1%E5%8F%B7%E9%87%8F)是一个同步对象，用于保持在0至指定最大值之间的一个计数值。

当线程完成一次对该semaphore对象的等待（wait）时，该计数值减一；

当线程完成一次对semaphore对象的释放（release）时，计数值加一。

当计数值为0，则线程等待该semaphore对象不再能成功直至该semaphore对象变成signaled状态。

具体的讲，在c++的实现中，如果当前信号量的计数值为0，那么执行“等待”操作的线程将会一直阻塞，直到计数大于 0，也就是其它线程执行了“释放”操作。

c++提供了两种信号量的实现：

1. `std::counting_semaphore`
2. `std::binary_semaphore`

其中，`binary_semaphore`只是`counting_semaphore`的一个特化：

```cpp
using binary_semaphore = counting_semaphore<1>;
```

代码示例：

```cpp
// 全局二元信号量对象
// 设置对象初始计数为 0
std::binary_semaphore smph_signal_main_to_thread{ 0 };
std::binary_semaphore smph_signal_thread_to_main{ 0 };

void thread_proc() {
    smph_signal_main_to_thread.acquire();
    std::cout << "[线程] 获得信号" << std::endl;

    std::this_thread::sleep_for(3s);

    std::cout << "[线程] 发送信号\n";
    // NOTE: 释放，原子的增加计数
    smph_signal_thread_to_main.release();
}

int main() {
    std::jthread thr_worker{ thread_proc };

    std::cout << "[主] 发送信号\n";
    smph_signal_main_to_thread.release();

    // NOTE: 等待, 原子的减少计数
    smph_signal_thread_to_main.acquire();
    std::cout << "[主] 获得信号\n";
}
```

信号量常用于发信/提醒而非互斥，通过初始化该信号量为 0 从而阻塞尝试`acquire()`的接收者，直至提醒者通过调用`release(n)`“发信”。

在此方面可把信号量当作条件变量的替代品，通常它有更好的性能。

假设我们有一个服务器，它只能处理有限数量的并发请求。为了防止服务器过载，我们可以使用信号量来限制并发请求的数量

```cpp
// 定义一个信号量，最大并发数为 3
std::counting_semaphore<3> semaphore{ 3 };

void handle_request(int request_id) {
    // 请求到达，尝试获取信号量
    std::cout << "进入 handle_request 尝试获取信号量\n";

    semaphore.acquire();

    std::cout << "成功获取信号量\n";

    // 此处延时三秒可以方便测试，会看到先输出 3 个“成功获取信号量”，因为只有三个线程能成功调用 acquire，剩余的会被阻塞
    std::this_thread::sleep_for(3s);

    // 模拟处理时间
    std::random_device rd;
    std::mt19937 gen{ rd() };
    std::uniform_int_distribution<> dis(1, 5);
    int processing_time = dis(gen);
    std::this_thread::sleep_for(std::chrono::seconds(processing_time));

    std::cout << std::format("请求 {} 已被处理\n", request_id);

    semaphore.release(); 
}

int main() {
    // 模拟 10 个并发请求
    std::vector<std::jthread> threads;
    for (int i = 0; i < 10; ++i) {
        threads.emplace_back(handle_request, i);
    }
}
```

简单的说2个概念

1. `counting_semaphore`是一个轻量同步原语，能控制对共享资源的访问。跟`std::mutex`不同，`counting_semaphore`不是互斥的，它是一个计数信号量，可以控制对共享资源的访问数量。至少允许`LeastMaxValue`个同时访问者
2. `binary_semaphore`是`std::counting_semaphore`的特化的别名，其`LeastMaxValue`为1

## 2. latch + barrier

这也是一种线程协调机制，和semaphore不同的是，latch和barrier允许任何数量的线程阻塞直至期待数量的线程到达。

通俗的讲，排队

1. semaphore是放人，放到n就不能再进了，除非里面的出来
2. latch是等人，等到n就能进了，不管里面的出来没有，是一种单次使用的线程屏障。
3. barrier是等人，等到n就能进了，然后重新复位，再等n

### 2.1 latch

latch维护着一个内部计数，且只能减少计数，无法增加计数。在创建对象的时候初始化计数器的值。

线程可以阻塞，直到 latch 对象的计数减少到零。由于无法增加计数，这使得latch成为一种单次使用的屏障。

```cpp
std::latch work_start{ 3 };

void work(){
    std::cout << "等待其它线程执行\n";
    work_start.wait(); // 等待计数为 0
    std::cout << "任务开始执行\n";
}

int main(){
    std::jthread thread{ work };
    std::this_thread::sleep_for(3s);
    std::cout << "休眠结束\n";
    work_start.count_down();  // 默认值是 1 减少计数 1
    work_start.count_down(2); // 传递参数 2 减少计数 2
}
```

latch的常用法就是划分任务执行的工作区间

```cpp
std::latch latch{ 10 };

void f(int id) {
    //todo.. 脑补任务
    std::this_thread::sleep_for(1s);
    std::cout << std::format("线程 {} 执行完任务，开始等待其它线程执行到此处\n", id);
    latch.arrive_and_wait(); // NOTE: 等价 = count_down(n); wait();
    std::cout << std::format("线程 {} 彻底退出函数\n", id);
}

int main() {
    std::vector<std::jthread> threads;
    for (int i = 0; i < 10; ++i) {
        threads.emplace_back(f,i);
    }
}
```

### 2.2 barrier

barrier是一种多次使用的屏障，它允许任意数量的线程阻塞，直到所有线程都到达barrier。在阶段完成之后将计数重置为构造时传递的值

```cpp
std::barrier barrier{ 10,
    [n = 1]()mutable noexcept {std::cout << "\t第" << n++ << "轮结束\n"; }
};

void f(int start, int end){
    for (int i = start; i <= end; ++i) {
        std::osyncstream{ std::cout } << i << ' '; 
        barrier.arrive_and_wait(); // 减少计数并等待 解除阻塞时就重置计数并调用函数对象
        
        std::this_thread::sleep_for(300ms);
    }
}

int main(){
    std::vector<std::jthread> threads;
    for (int i = 0; i < 10; ++i) {
        threads.emplace_back(f, i * 10 + 1, (i + 1) * 10);
    }
}
```

```text
1 21 11 31 41 51 61 71 81 91    第1轮结束
12 2 22 32 42 52 62 72 92 82    第2轮结束
13 63 73 33 23 53 83 93 43 3    第3轮结束
14 44 24 34 94 74 64 4 84 54    第4轮结束
5 95 15 45 75 25 55 65 35 85    第5轮结束
6 46 16 26 56 96 86 66 76 36    第6轮结束
47 17 57 97 87 67 77 7 27 37    第7轮结束
38 8 28 78 68 88 98 58 18 48    第8轮结束
9 39 29 69 89 99 59 19 79 49    第9轮结束
30 40 70 10 90 50 60 20 80 100  第10轮结束
```

利用的barrier给任务做分组, 需要注意的是，`lambda`表达式必须声明为`noexcept`，因为`std::barrier`要求其函数对象类型必须是不抛出异常的。

```cpp
std::is_nothrow_invocable_v<_Completion_function&> == true
```

当然如果你不是msvc的编译器，也无所谓，gcc + clang目前不会做检查
