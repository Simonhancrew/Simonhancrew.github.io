---
title: c++的lambda捕获-02
date: 2025-04-11 21:00:00 +0800
categories: [Blogging, c++]
tags: [writing]
---

单纯记录一点东西，代码都是cpp insight生成的

代码大概是

```cpp
#include <cstdio>
#include <iostream>

using namespace std;

struct test{
  int id_ = 0;
  auto get() {
  	auto func = [=]() {
    	cout << id_ << '\n';
   	};
    return func;
  }
};


int main()
{
  test t;
  auto res = t.get();
}
```

等价于

```cpp
#include <cstdio>
#include <iostream>

using namespace std;

struct test
{
  int id_ = 0;
  inline __lambda_9_16 get()
  {
        
    class __lambda_9_16
    {
      public: 
      inline /*constexpr */ void operator()() const
      {
        std::operator<<(std::cout.operator<<(__this->id_), '\n');
      }
      
      private: 
      test * __this;
      public: 
      // inline /*constexpr */ __lambda_9_16(__lambda_9_16 &&) noexcept = default;
      __lambda_9_16(test * _this)
      : __this{_this}
      {}
      
    };
    
    __lambda_9_16 func = __lambda_9_16{this} /* NRVO variable */;
    return func;
  }
  
  // inline constexpr test() noexcept = default;
};


int main()
{
  test t = test();
  __lambda_9_16 res = t.get();
  return 0;
}
```

这个是c++20以前的，`=`还是会默认捕获this，20开始这个就编译不过了，会默认不捕获this

另外msvc有一个bug

```cpp
struct S {
  static void Method();
  void Method(int x);
  void Test() {
    auto f = [/*this*/] { S::Method(); };
  }
};
```

这里一定要带一个this，不然会有warning，这个是[msvc的一个bug](https://developercommunity.visualstudio.com/t/bogus-warning-c4573-for-static-method-with-same-na-2/950213)，修复在vs2019直接带上

1. `/Zc:lambda`
2. `/std:c++latest` 

虽然理论上他解析出来的应该类似是下面的，应该不需要this的才是

```cpp
struct S
{
  static void Method();
  
  void Method(int x);
  
  inline void Test()
  {
        
    class __lambda_10_14
    {
      public: 
      inline /*constexpr */ void operator()() const
      {
        Method();
      }
      
      using retType_10_14 = auto (*)() -> void;
      inline constexpr operator retType_10_14 () const noexcept
      {
        return __invoke;
      };
      
      private: 
      static inline /*constexpr */ void __invoke()
      {
        __lambda_10_14{}.operator()();
      }
      
      
    };
    
    __lambda_10_14 f = __lambda_10_14{};
  }
  
};
```
