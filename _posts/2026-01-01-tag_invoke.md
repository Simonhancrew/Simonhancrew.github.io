---
title: tag_invoke
date: 2026-01-01 23:23:00 +0800
categories: [Blogging, c++, tag invoke]
tags: [writing]
---

在看libunifex的代码，里面用了比较多的tag_invoke来连接receiver和sender，主要记一下tag_invoke的用法，太jb傻逼难读了。

### 1. 为什么需要它？（痛点）

在 C++ 中，如果你想让不同的类支持同一个操作（比如 `start` 一个异步任务），通常有两种做法：

*   **做法 A：虚函数（多态）**。
    *   缺点：有虚函数表指针开销，不能在编译期优化（内联），且你无法给 `std::vector` 这种现有的类添加虚函数。
*   **做法 B：普通的函数重载（ADL 机制）**。
    *   缺点：查找规则极其复杂，容易起冲突，报错信息难以理解。

**`tag_invoke` 诞生了：** 它要把所有这些操作统一到一个入口下。

---

### 2. `tag_invoke` 的直观理解

你可以把 `tag_invoke` 想象成一个**中转站**。

假设我们要实现一个 `connect` 操作：
1.  **定义一个标签（Tag）**：`struct connect_t {};`
2.  **定义一个统一入口**：不管你是谁，只要你想调用 `connect`，你就调用 `tag_invoke(connect_t{}, obj, ...)`。
3.  **用户实现**：具体的类只需要在自己的命名空间里实现一个 `tag_invoke` 函数，第一个参数是 `connect_t`。

---

### 3. 代码长什么样？

我们可以拆解成三个角色：

#### 第一步：库的作者定义“操作”（定义插座）
```cpp
// 在 unifex 库里
struct connect_t {
    // 这是一个简化版的入口
    template<typename S, typename R>
    auto operator()(S&& s, R&& r) const {
        // 它会去寻找匹配的 tag_invoke 函数
        return tag_invoke(*this, (S&&)s, (R&&)r);
    }
};

inline constexpr connect_t connect{}; // 定义全局常量
```

#### 第二步：你定义自己的类（实现插头）
```cpp
struct MyReceiver {
    // Receiver 的实现
};

struct MySender {
    // 注意：这是一个友元函数，第一个参数是标签类型
    friend auto tag_invoke(connect_t, MySender& self, MyReceiver& r) {
        std::cout << "MySender connected!\n";
        return MyOperationState{};
    }
};
```

#### 第三步：使用方（插上电源）
```cpp
MySender s;
MyReceiver r;

// 统一调用方式
unifex::connect(s, r); 
// 实际上内部执行了 tag_invoke(connect_t{}, s, r)
```

---

### 4. 为什么 `libunifex` 疯狂使用它？

`tag_invoke`主要优势：

1.  **非侵入性 (Non-intrusive)**：
    你可以在不修改原始类定义的情况下，为第三方库的类添加功能。只要定义一个对应的 `tag_invoke` 重载即可。
  
2.  **完美的泛型支持**：
    在模板代码里，我只需要写 `unifex::connect(sender, receiver)`，编译器就能自动根据 `sender` 的类型找到正确的实现。如果 `sender` 没实现这个操作，编译报错会非常明确。

3.  **零成本抽象 (Zero-overhead)**：
    它在编译期就完成了分发，没有虚函数的性能损失，编译器可以轻松地进行内联优化。

---

### 总结

`tag_invoke` 就像是 C++ 里的 **Trait (Rust)** 或 **Interface (Go)** 的静态实现版本。它把“**做什么操作**”（Tag）和“**谁来做**”（Type）解耦了，然后通过一个全局函数把它们撮合在一起。

这里就**把逻辑拆散到了各个 `tag_invoke` 的重载里，而不是集中在一个类的方法里。** 你需要通过 Tag 去寻找真正的逻辑实现。
