---
title: shared ptr
date: 2024-04-09 14:10:00 +0800
categories: [Blogging, c++]
tags: [writing]
---

shared_ptr内部主要有两个点，第一个是contrl block，第二个是内部储存的类空间

ctrl block是在堆上创建的，内部会有一个atomic变量来做引用计数的增减, 因此，shared_ptr的copy其实是线程安全的，这也意味着修改指向内容的操作不是线程安全的。

这里谈到第二个点，为什么推荐使用make_shared，原因之一是RAII问题，new构造 + shared_ptr构造可能不在同一时期，中间抛异常的话可能导致内存泄漏

```cpp
vector<shared_ptr<A>> rec;
rec.emplace_back(new A);
```

第二个原因就是llvm的实现下，new一个shared ptr看起来会new两次，分别ctrl + T, 但是make shared看起来会直接要一片大内存，本质上其实只new了一次

```cpp
template <class _Tp, class... _Args, __enable_if_t<!is_array<_Tp>::value, int> = 0>
_LIBCPP_HIDE_FROM_ABI shared_ptr<_Tp> make_shared(_Args&&... __args) {
return std::allocate_shared<_Tp>(allocator<_Tp>(), std::forward<_Args>(__args)...);
}
```

具体看的话`__allocation_guard<_ControlBlockAllocator> __guard(__a, 1);`先构造了一片内存，随后placement new

```cpp
//
// std::allocate_shared and std::make_shared
//
template <class _Tp, class _Alloc, class... _Args, __enable_if_t<!is_array<_Tp>::value, int> = 0>
_LIBCPP_HIDE_FROM_ABI shared_ptr<_Tp> allocate_shared(const _Alloc& __a, _Args&&... __args) {
  using _ControlBlock          = __shared_ptr_emplace<_Tp, _Alloc>;
  using _ControlBlockAllocator = typename __allocator_traits_rebind<_Alloc, _ControlBlock>::type;
  __allocation_guard<_ControlBlockAllocator> __guard(__a, 1);
  ::new ((void*)std::addressof(__guard.__get())) _ControlBlock(__a, std::forward<_Args>(__args)...);
  auto __control_block = __guard.__release_ptr();
  return shared_ptr<_Tp>::__create_with_control_block(*(*__control_block).__get_elem(), std::addressof(*__control_block));
}
```

内部`__create_with_control_block`没有什么内存分配

```cpp
template <class _Yp, class _CntrlBlk>
_LIBCPP_HIDE_FROM_ABI static shared_ptr<_Tp> __create_with_control_block(
    _Yp* __p, _CntrlBlk* __cntrl) _NOEXCEPT {
  shared_ptr<_Tp> __r;
  __r.__ptr_ = __p;
  __r.__cntrl_ = __cntrl;
  __r.__enable_weak_this(__r.__ptr_, __r.__ptr_);
  return __r;
}
```

`shared_ptr`的构造函数，看下msvc的, 要区分array type和scalar type

```cpp
template <class _Ux,
    enable_if_t<conjunction_v<conditional_t<is_array_v<_Ty>, _Can_array_delete<_Ux>, _Can_scalar_delete<_Ux>>,
                    _SP_convertible<_Ux, _Ty>>,
        int> = 0>
explicit shared_ptr(_Ux* _Px) { // construct shared_ptr object that owns _Px
    if constexpr (is_array_v<_Ty>) {
        _Setpd(_Px, default_delete<_Ux[]>{});
    } else {
        _Temporary_owner<_Ux> _Owner(_Px);
        _Set_ptr_rep_and_enable_shared(_Owner._Ptr, new _Ref_count<_Ux>(_Owner._Ptr));
        _Owner._Ptr = nullptr;
    }
}
```

这个`_Ref_count`实际上就是一个ctrl block, _Atomic_counter_t虽然是unsigned long，但是实际是用`_InterlockedIncrement`来操作数的，也是原子操作。

```cpp
class __declspec(novtable) _Ref_count_base { // common code for reference counting
private:
#ifdef _M_CEE_PURE
    // permanent workaround to avoid mentioning _purecall in msvcurt.lib, ptrustu.lib, or other support libs
    virtual void _Destroy() noexcept {
        _CSTD abort();
    }

    virtual void _Delete_this() noexcept {
        _CSTD abort();
    }
#else // ^^^ defined(_M_CEE_PURE) / !defined(_M_CEE_PURE) vvv
    virtual void _Destroy() noexcept     = 0; // destroy managed resource
    virtual void _Delete_this() noexcept = 0; // destroy self
#endif // ^^^ !defined(_M_CEE_PURE) ^^^

    _Atomic_counter_t _Uses  = 1;
    _Atomic_counter_t _Weaks = 1;

protected:
    constexpr _Ref_count_base() noexcept = default; // non-atomic initializations

public:
    _Ref_count_base(const _Ref_count_base&)            = delete;
    _Ref_count_base& operator=(const _Ref_count_base&) = delete;

    virtual ~_Ref_count_base() noexcept {} // TRANSITION, should be non-virtual

    bool _Incref_nz() noexcept { // increment use count if not zero, return true if successful
        auto& _Volatile_uses = reinterpret_cast<volatile long&>(_Uses);
#ifdef _M_CEE_PURE
        long _Count = *_Atomic_address_as<const long>(&_Volatile_uses);
#else
        long _Count = __iso_volatile_load32(reinterpret_cast<volatile int*>(&_Volatile_uses));
#endif
        while (_Count != 0) {
            const long _Old_value = _INTRIN_RELAXED(_InterlockedCompareExchange)(&_Volatile_uses, _Count + 1, _Count);
            if (_Old_value == _Count) {
                return true;
            }

            _Count = _Old_value;
        }

        return false;
    }

    void _Incref() noexcept { // increment use count
        _MT_INCR(_Uses);
    }

    void _Incwref() noexcept { // increment weak reference count
        _MT_INCR(_Weaks);
    }

    void _Decref() noexcept { // decrement use count
        if (_MT_DECR(_Uses) == 0) {
            _Destroy();
            _Decwref();
        }
    }

    void _Decwref() noexcept { // decrement weak reference count
        if (_MT_DECR(_Weaks) == 0) {
            _Delete_this();
        }
    }

    long _Use_count() const noexcept {
        return static_cast<long>(_Uses);
    }

    virtual void* _Get_deleter(const type_info&) const noexcept {
        return nullptr;
    }
};
```

`_Set_ptr_rep_and_enable_shared`可以看到有针对enable_shared_from_this的处理

```cpp
template <class _Ux>
void _Set_ptr_rep_and_enable_shared(_Ux* const _Px, _Ref_count_base* const _Rx) noexcept { // take ownership of _Px
    this->_Ptr = _Px;
    this->_Rep = _Rx;
    if constexpr (conjunction_v<negation<is_array<_Ty>>, negation<is_volatile<_Ux>>, _Can_enable_shared<_Ux>>) {
        if (_Px && _Px->_Wptr.expired()) {
            _Px->_Wptr = shared_ptr<remove_cv_t<_Ux>>(*this, const_cast<remove_cv_t<_Ux>*>(_Px));
        }
    }
}

void _Set_ptr_rep_and_enable_shared(nullptr_t, _Ref_count_base* const _Rx) noexcept { // take ownership of nullptr
    this->_Ptr = nullptr;
    this->_Rep = _Rx;
}
```

`_Can_enable_shared`就是一个简单的判断有没有继承enable_shared_from_this

可以看到ctrl block和shared ptr其实是分开做模板的， 这就可以有一个比较有意思的点, 下面的基类A没有virtual析构函数，但是也能正常析构，这是因为ctrl block里会拿到子类型，然后析构的时候会调用子类型的析构函数

```cpp
#include <iostream>

using namespace std;

class A {
public:
  ~A() { cout << "destroy A\n"; }
};

class B : public A {
public:
  ~B() { cout << "destroy B\n"; }
};

int main() {
  std::shared_ptr<A> ptr = std::make_shared<B>();
  ptr.reset();
}
```

也可以玩点花活, 搞个function做类型擦除的deleter。

```cpp
#include <functional>
#include <iostream>


using namespace std;

class A {
public:
  ~A() { cout << "destroy A\n"; }
};

class B : public A {
public:
  ~B() { cout << "destroy B\n"; }
};

template <typename T> class my_ptr {
public:
  template <typename P> my_ptr(P *p) {
    data_ = p;
    deleter_ = [p]() { delete p; };
  }

  ~my_ptr() {
    if (data_ != nullptr) {
      deleter_();
    }
  }

  void reset() {
    data_ = nullptr;
    deleter_();
  }

private:
  std::function<void()> deleter_;
  T *data_;
};

int main() {
  my_ptr<A> ptr(new B);
  ptr.reset();
}
```

### 一些要注意的点

1. 尽量用make_shared
2. shared_ptr可以自定义deleter，而且比unique_ptr好的点是可以直接写成lambda, 但是这样就不能用make_shared了
3. 不用拿裸指针创建多个shared_ptr
4. 类返回this->shared_ptr的话，记得enale_shared_from_this + shared_from_this, 因为他本质也是个指针，跟3一样
5. shared_ptr的copy是线程安全的，但是修改指向内容的操作不是线程安全的

### REF

1. [shared_ptr线程安全](https://zhuanlan.zhihu.com/p/416289479)
