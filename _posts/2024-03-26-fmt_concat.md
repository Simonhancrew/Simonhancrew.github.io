---
title: ostringstream的性能问题"
date: 2024-03-26 14:10:00 +0800
categories: [Blogging, cpp]
tags: [writing]
---

其实最悲伤的一点是，我在之前的代码里大量用到了ostringstream，当时没有意识到这玩意其实在多线程下可能是有副作用的。

查资料的时候发现，ostringstream在构造的时候有加锁行为，在解释一些宽字符串（如汉字）的时候，依赖执行环境的本地化策略，一个可执行文件在运行前是无法确定这些转换策略的，所以ostringstream在构造的时候需要通过 std::locale()来获取本地化策略，std::locale()内其实是拷贝了全局的本地化策略，同时系统允许对本地化策略进行更改和重新设置。特别是使用ostringstream比较频繁的代码，可能会导致频繁的加锁和解锁操作，这个开销是很大的。

```cpp
class ios_base {
public:
    locale
  getloc() const
  { return _M_ios_locale; }
​
  /**
    *  @brief  Locale access
    *  @return  A reference to the current locale.
    *
    *  Like getloc above, but returns a reference instead of
    *  generating a copy.
  */
  const locale&
  _M_getloc() const
  { return _M_ios_locale; }
protected:
    locale        _M_ios_locale;

}


const locale&
locale::operator=(const locale& __other) throw()
{
  __other._M_impl->_M_add_reference();
  _M_impl->_M_remove_reference();
  _M_impl = __other._M_impl;
  return *this;
}
​
locale::locale(const locale& __other) throw()
: _M_impl(__other._M_impl)
{ _M_impl->_M_add_reference(); }
​
locale::~locale() throw()
{ _M_impl->_M_remove_reference(); }

 class locale::_Impl
  {
    ...
    void
    _M_add_reference() throw()
    { __gnu_cxx::__atomic_add_dispatch(&_M_refcount, 1); }
​
    void
    _M_remove_reference() throw()
    {
      // Be race-detector-friendly.  For more info see bits/c++config.
      _GLIBCXX_SYNCHRONIZATION_HAPPENS_BEFORE(&_M_refcount);
      if (__gnu_cxx::__exchange_and_add_dispatch(&_M_refcount, -1) == 1)
    {
          _GLIBCXX_SYNCHRONIZATION_HAPPENS_AFTER(&_M_refcount);
      __try
        { delete this; }
      __catch(...)
        { }
    }
    }
  }
```

### REF

1. [C++ std::ostringstream 多线程性能问题探究](https://zhuanlan.zhihu.com/p/655076746)
2. [ostringstream多线程下性能问题分析](https://chys.info/blog/2017-11-06-ostringstream-performance)

