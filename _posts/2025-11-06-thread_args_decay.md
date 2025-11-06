---
title: std::thread arguments decay issue
date: 2025-11-06 21:10:00 +0800
categories: [Blogging, c++]
tags: [writing]
---

std::thread在传递参数的时候的decay的代码走读一下

```cpp
template <class _Fn, class... _Args, enable_if_t<!is_same_v<_Remove_cvref_t<_Fn>, thread>, int> = 0>
_NODISCARD_CTOR_THREAD explicit thread(_Fn&& _Fx, _Args&&... _Ax) {
    _Start(_STD forward<_Fn>(_Fx), _STD forward<_Args>(_Ax)...);
}

template <class _Fn, class... _Args>
void _Start(_Fn&& _Fx, _Args&&... _Ax) {
    using _Tuple                 = tuple<decay_t<_Fn>, decay_t<_Args>...>;
    auto _Decay_copied           = _STD make_unique<_Tuple>(_STD forward<_Fn>(_Fx), _STD forward<_Args>(_Ax)...);
    constexpr auto _Invoker_proc = _Get_invoke<_Tuple>(make_index_sequence<1 + sizeof...(_Args)>{});

    _Thr._Hnd =
        reinterpret_cast<void*>(_CSTD _beginthreadex(nullptr, 0, _Invoker_proc, _Decay_copied.get(), 0, &_Thr._Id));

    if (_Thr._Hnd) { // ownership transferred to the thread
        (void) _Decay_copied.release();
    } else { // failed to start thread
        _Thr._Id = 0;
        _Throw_Cpp_error(_RESOURCE_UNAVAILABLE_TRY_AGAIN);
    }
}
```

这里可以比较清楚的看到，传递给thread的参数会经过decay处理，然后再存储到tuple里，最后通过invoke调用

这个decay的过程是

```cpp
_EXPORT_STD template <class _Ty>
struct decay { // determines decayed version of _Ty
    using _Ty1 = remove_reference_t<_Ty>;
    using _Ty2 = typename _Select<is_function_v<_Ty1>>::template _Apply<add_pointer<_Ty1>, remove_cv<_Ty1>>;
    using type = typename _Select<is_array_v<_Ty1>>::template _Apply<add_pointer<remove_extent_t<_Ty1>>, _Ty2>::type;
};

_EXPORT_STD template <class _Ty>
using decay_t = typename decay<_Ty>::type;
```

翻了一下remove ref跟remove cv的实现，都是比较直观的类型萃取，靠模板偏特化实现的，没什么复杂的

Select这里就是一个简单的条件选择模板

最后decay的作用是3点

1. 如果是引用类型，去掉引用
2. 如果是函数类型，变成函数指针
3. 如果是数组类型，变成指向数组元素类型的指针

这种设计应该是故意的，如果允许直接引用，比较容易造成悬垂，默认就拷贝一下，如果用户想要引用语义，可以传递std::ref/std::cref

这个ref就套了一个reference_wrapper

```cpp
_EXPORT_STD template <class _Ty>
_NODISCARD _CONSTEXPR20 reference_wrapper<_Ty> ref(_Ty& _Val) noexcept {
    return reference_wrapper<_Ty>(_Val);
}


_EXPORT_STD template <class _Ty>
class reference_wrapper
#if !_HAS_CXX20
    : public _Weak_types<_Ty>
#endif // !_HAS_CXX20
{
public:
    static_assert(is_object_v<_Ty> || is_function_v<_Ty>,
        "reference_wrapper<T> requires T to be an object type or a function type.");

    using type = _Ty;

    template <class _Uty, enable_if_t<conjunction_v<negation<is_same<_Remove_cvref_t<_Uty>, reference_wrapper>>,
                                          _Refwrap_has_ctor_from<_Ty, _Uty>>,
                              int> = 0>
    _CONSTEXPR20 reference_wrapper(_Uty&& _Val) noexcept(
        noexcept(_STD _Refwrap_ctor_fun<_Ty>(_STD declval<_Uty>()))) { // qualified: avoid ADL, handle incomplete types
        _Ty& _Ref = static_cast<_Uty&&>(_Val);
        _Ptr      = _STD addressof(_Ref);
    }

    _CONSTEXPR20 operator _Ty&() const noexcept {
        return *_Ptr;
    }

    _NODISCARD _CONSTEXPR20 _Ty& get() const noexcept {
        return *_Ptr;
    }

private:
    _Ty* _Ptr{};

public:
    template <class... _Types>
    _CONSTEXPR20 auto operator()(_Types&&... _Args) const
        noexcept(noexcept(_STD invoke(*_Ptr, static_cast<_Types&&>(_Args)...))) //
        -> decltype(_STD invoke(*_Ptr, static_cast<_Types&&>(_Args)...)) {
        return _STD invoke(*_Ptr, static_cast<_Types&&>(_Args)...);
    }
};
```

实际上是拿到了引用的地址，当最后需要实际调用函数的时候是

```cpp
f(int& arg);
```

实际这里给arg的是reference_wrapper<int>，这里会通过`operator int&()`的隐式转换转换成`int&`传递给`f`
