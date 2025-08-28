---
title: iterator的问题
date: 2025-08-28 16:44:14 +0800
categories: [Blogging, iterator, c++, bug]
tags: [writing]
---

### end()能++吗

```
The behavior is undefined if the specified sequence of increments or decrements would require that a non-incrementable iterator (such as the past-the-end iterator) is incremented, or that a non-decrementable iterator (such as the front iterator or the singular iterator) is decremented.
```

首先结论是最好别这么搞，这里是个UB

具体的看一眼代码，以unordered_map的iter为例子，map取end()的代码

```cpp
typedef __hash_table<__value_type, __hasher,
                        __key_equal,  __allocator_type>   __table;

__table __table_;
iterator       end() _NOEXCEPT          {return __table_.end();}
```

随后看这个iter怎么被构造的

```cpp
template <class _Tp, class _Hash, class _Equal, class _Alloc>
inline
typename __hash_table<_Tp, _Hash, _Equal, _Alloc>::iterator
__hash_table<_Tp, _Hash, _Equal, _Alloc>::end() _NOEXCEPT
{
    return iterator(nullptr);
}

template <class _NodePtr>
class _LIBCPP_TEMPLATE_VIS __hash_iterator
{
    typedef __hash_node_types<_NodePtr> _NodeTypes;
    typedef _NodePtr                            __node_pointer;
    typedef typename _NodeTypes::__next_pointer __next_pointer;

    __next_pointer            __node_;
    _LIBCPP_INLINE_VISIBILITY
    explicit __hash_iterator(__next_pointer __node) _NOEXCEPT
        : __node_(__node)
        {
        }
}
```

这里看到__node_是个nullptr，他的++形如

```cpp
_LIBCPP_INLINE_VISIBILITY
__hash_iterator& operator++() {
    __node_ = __node_->__next_;
    return *this;
}
```

再看list的话，其实是支持这么搞的，因为list的end是个自指向的iter node，prev跟next都指向自己。
