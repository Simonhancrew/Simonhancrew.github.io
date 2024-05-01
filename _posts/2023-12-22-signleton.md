---
title: c++11 singleton"
date: 2023-12-22 14:10:00 +0800
categories: [Blogging, c++]
tags: [writing]
---

c++11 singleton

在C++11标准中，局部静态变量的初始化是线程安全的，这就意味着在多线程环境中可以保障局部静态变量只会被初始化一次，而不会引发竞态条件问题。

因此11支持的一个比较简单的写法， 这种由编译器保证的单例

```c++
Singleton& get() {
  static Singleton instance;
  return instance;
}
// 或者
class Singleton {
public:
  static Singleton &instance() {
    static Singleton s;
    return s;
  }

  // 删除复制构造函数和赋值操作符
  Singleton(const Singleton &) = delete;
  Singleton &operator=(const Singleton &) = delete;

protected:
  Singleton() {}
  ~Singleton() {}
};
```

返回指针的话

```cpp
class Singleton {
 public:
  Singleton(Singleton const&) = delete;
  Singleton& operator=(Singleton const&) = delete;
  ~Singleton() {}
  // Set value has to be thread-safe.
  void set_value(int value)
  {
    std::lock_guard<std::mutex> lock(m_mutex);
    m_value = value;
  }
  // We don't want get value to be thread-safe in our case.
  int get_value() const { return m_value; }
  static Singleton* get_instance()
  {
    // Static local variable initialization is thread-safe
    // and will be initialized only once.
    static Singleton instance{};
    return &instance;
  }

 private:
  explicit Singleton() : m_value{0} {}
  std::mutex m_mutex;
  int m_value;
};
```

另外也可以依赖once_flag

```cpp
template<class T> 
class Resource
{
  Resource<T>(const Resource<T>&) = delete;
  Resource<T>& operator=(const Resource<T>&) = delete;
  static unique_ptr<Resource<T>> m_ins;
  static once_flag m_once;
  Resource<T>() = default;

public : 
  virtual ~Resource<T>() = default;
  static Resource<T>& getInstance() {
      std::call_once(m_once, []() {
          m_ins.reset(new Resource<T>);
      });
      return *m_ins.get();
  }
};
```

## REF

1. [The Singleton](https://www.modernescpp.com/index.php/creational-patterns-singleton/)

2. [Thread-Safe Initialization of a Singleton](https://www.modernescpp.com/index.php/thread-safe-initialization-of-a-singleton/)

3. [深入分析C++全局变量初始化](https://zhuanlan.zhihu.com/p/664349537)
