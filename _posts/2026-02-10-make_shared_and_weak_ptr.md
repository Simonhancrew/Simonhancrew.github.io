---
title: make_shared和weak_ptr的使用
date: 2026-02-10 21:57:00 +0800
categories: [Blogging, c++]
tags: [writing]
---

今天读msvc的c++标准库实现发现的，有一个现象是make_shared构造的对象和直接new出来的对象，在内存布局上是稍微有点不一样的。

直接剧透的话，他们的区别重点在于cntrl块的布局上是不同的，这个可能在weak_ptr存在的情况下影响内存的最后释放时机

make_shared的对象的cntrl块的layout是

```
 --------------------------------------------------------------
 |                |               |                |          |
 |   object       |   ref count   |   weak count   |   ...    |
 |                |               |                |          | 
 --------------------------------------------------------------
```

而new出来的对象的layout是

```
 --------------------------------------------------------------
 |   ptr       |   ref count   |   weak count   |   ...    |
 --------------------------------------------------------------
      |
      |         ---------
      -------> |  object |
                ---------
```

make_shared构造的对象的析构可能稍微复杂一点，直到所有的weak_ptr都被销毁，这里分配的内存才可能被释放掉

假设有一个这样的代码

```cpp
#include <iostream>
#include <memory>

struct LargeObject {
  int data[1000];
  LargeObject() { std::cout << "LargeObject construct called\n"; }
  ~LargeObject() { std::cout << "LargeObject deconstruct called\n"; }
};

void *operator new(size_t count) {
  std::cout << "allocating " << count << " bytes\n";
  return malloc(count);
}

void operator delete(void *ptr) noexcept {
  std::cout << "deallocating memory\n";
  free(ptr);
}

std::weak_ptr<LargeObject> GetweakPtrByMakeShared() {
  auto shared_ptr = std::make_shared<LargeObject>();
  auto weak_ptr = shared_ptr;
  return weak_ptr;
}

std::weak_ptr<LargeObject> GetweakPtrByNew() {
  auto shared_ptr = std::shared_ptr<LargeObject>(new LargeObject());
  auto weak_ptr = shared_ptr;
  return weak_ptr;
}

int main() {
  auto weak_ptr1 = GetweakPtrByMakeShared();
  std::cout << "weak_ptr1 use_count: " << weak_ptr1.use_count()
            << std::endl; // 输出 1>

  std::cout << "-----------------------------\n";
  // auto weak_ptr2 = GetweakPtrByNew();
  // std::cout << "weak_ptr2 use_count: " << weak_ptr2.use_count()
  //           << std::endl; // 输出 1>
  // std::cout << "-----------------------------\n";
  std::cout << "out of main scope\n";
}
```

这里的输出中，make_shared构造的对象会在main函数结束时才执行内存的释放, 原因是这里的layout不同

当使用make_shared的时候,这里的代码是

```cpp
_NODISCARD_SMART_PTR_ALLOC shared_ptr<_Ty> make_shared(_Types&&... _Args) { // make a shared_ptr to non-array object
    const auto _Rx = new _Ref_count_obj2<_Ty>(_STD forward<_Types>(_Args)...);
    shared_ptr<_Ty> _Ret;
    _Ret._Set_ptr_rep_and_enable_shared(_STD addressof(_Rx->_Storage._Value), _Rx);
    return _Ret;
}


template <class _Ty>
class _Ref_count_obj2 : public _Ref_count_base { // handle reference counting for object in control block, no allocator
public:
    template <class... _Types>
    explicit _Ref_count_obj2(_Types&&... _Args) : _Ref_count_base() {
#if _HAS_CXX20
        if constexpr (sizeof...(_Types) == 1 && (is_same_v<_For_overwrite_tag, remove_cvref_t<_Types>> && ...)) {
            _STD _Default_construct_in_place(_Storage._Value);
            ((void) _Args, ...);
        } else
#endif // _HAS_CXX20
        {
            _STD _Construct_in_place(_Storage._Value, _STD forward<_Types>(_Args)...);
        }
    }

    ~_Ref_count_obj2() noexcept override { // TRANSITION, should be non-virtual
        // nothing to do, _Storage._Value was already destroyed in _Destroy

        // N4950 [class.dtor]/7:
        // "A defaulted destructor for a class X is defined as deleted if:
        // X is a union-like class that has a variant member with a non-trivial destructor"
    }

    union {
        _Wrap<remove_cv_t<_Ty>> _Storage;
    };

private:
    void _Destroy() noexcept override { // destroy managed resource
        _STD _Destroy_in_place(_Storage._Value);
    }

    void _Delete_this() noexcept override { // destroy self
        delete this;
    }
};
```


这里的ref_count_obj2是一个类，里面有一个union成员变量，里面存储了真正的对象，这个对象的生命周期是由ref_count_obj2来管理的，所以当ref_count_obj2被销毁的时候，才会去回收这块内存。

当直接用new出来的对象构造shared_ptr的时候，这里的代码是

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
```

这里的是一个_Ref_count类，这个类的实现非常简单，就是存一个指针，因此这里的存储空间其实是跟cntrl分离的

```cpp
template <class _Ty>
class _Ref_count : public _Ref_count_base { // handle reference counting for pointer without deleter
public:
    explicit _Ref_count(_Ty* _Px) : _Ref_count_base(), _Ptr(_Px) {}

private:
    void _Destroy() noexcept override { // destroy managed resource
        delete _Ptr;
    }

    void _Delete_this() noexcept override { // destroy self
        delete this;
    }

    _Ty* _Ptr;
};
```

所以析构的时候的行为稍微有点不同

```cpp
~shared_ptr() noexcept { // release resource
    this->_Decref();
}
void _Decref() noexcept { // decrement reference count
    if (_Rep) {
        _Rep->_Decref();
    }
}
void _Decref() noexcept { // decrement use count
    if (_MT_DECR(_Uses) == 0) {
        _Destroy();
        _Decwref();
    }
}
```

最后这里的两个destroy,_Decwref的实现比较简单, 都是把weak计数减到0之后销毁refcount的cntrl块

不同的点在于shared cnt清零的时候的处理

1. make_shared出来的对象只执行简单的析构，object的回收要等到weak_ptr的计数也清零之后，执行cntrl块的析构来完整回收object的内存
2. new出来的对象在shared cnt清零的时候就会执行delete来回收内存了，因为他直接是一个指针，指向new出来的内存，直接delete ptr就可以回收内存

```cpp
// for make_shared
void _Destroy() noexcept override { // destroy managed resource
    _STD _Destroy_in_place(_Storage._Value);
}
// for new construct
void _Destroy() noexcept override { // destroy managed resource
    delete _Ptr;
}

void _Decwref() noexcept { // decrement weak reference count
    if (_MT_DECR(_Weaks) == 0) {
        _Delete_this();
    }
}
```
