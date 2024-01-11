---
layout: post
title:  "c++11 singleton"
category: c++
date:   2023-12-22
---

c++11 singleton

11支持的一个比较简单的写法， 这种由编译器保证的单例

```c++
static Singleton& get() {
  static Singleton instance;
  return instance;
}
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
