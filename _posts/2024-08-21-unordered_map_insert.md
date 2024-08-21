---
title: hashmap的复杂度分析
date: 2024-08-21 19:54:00 +0800
categories: [Blogging, c++, data-structure, std::unordered_map]
tags: [writing]
---

看下msvc下实现的`std::unordered_map`, 写的有点复杂，但是思路很简单

1. 针对key type，计算他的hash值，拿到index
2. 计算元素在hash桶中的索引，如果超出了范围要缩小`const size_type _Bucket = _Hashval & _Mask;`
3. 拿到了hashtable的index，数组每个点存的是一个链表，新插入的点都是头插法进入链表
4. 如果出现了较多的hash冲突，一个链表会变得过长，这个时候的效率会急剧下降，所以需要进行rehash
   1. 负载因子（load_factor），是hashtable的元素个数与hashtable的桶数之间比值；
   2. 最大负载因子（max_load_factor），是负载因子的上限
5. 随后就是进行load factor的降低，扩容+新的桶，将之前的元素重新hash到现在的桶中去

因此，insert之后导致了rehash，整体的数据其实是在变化的，迭代器失效，所以其实线程分离的读写是要上锁的

`unordered_map`提供了参数来观测当前的桶跟load_factor的值

```cpp
bucket_count()
load_factor()
```

## 具体的代码

从构造函数出发

```cpp
_EXPORT_STD template <class _Kty, class _Ty, class _Hasher = hash<_Kty>, class _Keyeq = equal_to<_Kty>,
    class _Alloc = allocator<pair<const _Kty, _Ty>>>
class unordered_map : public _Hash<_Umap_traits<_Kty, _Ty, _Uhash_compare<_Kty, _Hasher, _Keyeq>, _Alloc, false>> {
    // hash table of {key, mapped} values, unique keys
public:
    static_assert(!_ENFORCE_MATCHING_ALLOCATORS || is_same_v<pair<const _Kty, _Ty>, typename _Alloc::value_type>,
        _MISMATCHED_ALLOCATOR_MESSAGE("unordered_map<Key, Value, Hasher, Eq, Allocator>", "pair<const Key, Value>"));
    static_assert(is_object_v<_Kty>, "The C++ Standard forbids containers of non-object types "
                                     "because of [container.requirements].");

private:
    using _Mytraits      = _Uhash_compare<_Kty, _Hasher, _Keyeq>;
    using _Mybase        = _Hash<_Umap_traits<_Kty, _Ty, _Mytraits, _Alloc, false>>;
    using _Alnode        = typename _Mybase::_Alnode;
    using _Alnode_traits = typename _Mybase::_Alnode_traits;
    using _Nodeptr       = typename _Mybase::_Nodeptr;
    using _Key_compare   = typename _Mybase::_Key_compare;

public:
    using hasher      = _Hasher;
    using key_type    = _Kty;
    using mapped_type = _Ty;
    using key_equal   = _Keyeq;

    using value_type      = pair<const _Kty, _Ty>;
    using allocator_type  = typename _Mybase::allocator_type;
    using size_type       = typename _Mybase::size_type;
    using difference_type = typename _Mybase::difference_type;
    using pointer         = typename _Mybase::pointer;
    using const_pointer   = typename _Mybase::const_pointer;
    using reference       = value_type&;
    using const_reference = const value_type&;
    using iterator        = typename _Mybase::iterator;
    using const_iterator  = typename _Mybase::const_iterator;

    using local_iterator       = typename _Mybase::iterator;
    using const_local_iterator = typename _Mybase::const_iterator;

#if _HAS_CXX17
    using insert_return_type = _Insert_return_type<iterator, typename _Mybase::node_type>;
#endif // _HAS_CXX17

    unordered_map() : _Mybase(_Key_compare(), allocator_type()) {}
}
```

这里看，`unordered_map`是从`_Hash`继承而来，这里的`_Hash`是一个模板类，`_Hash<_Umap_traits<_Kty, _Ty, _Uhash_compare<_Kty, _Hasher, _Keyeq>, _Alloc, false>>`

`_Hash`的模板参数, `_Umap_traits`是一个模板结构体，定义如下

```cpp
_STD_BEGIN
template <class _Kty, // key type
    class _Ty, // mapped type
    class _Tr, // comparator predicate type
    class _Alloc, // actual allocator type (should be value allocator)
    bool _Mfl> // true if multiple equivalent keys are permitted
class _Umap_traits : public _Tr { // traits required to make _Hash behave like a map
public:
    using key_type            = _Kty;
    using value_type          = pair<const _Kty, _Ty>;
    using _Mutable_value_type = pair<_Kty, _Ty>;
    using key_compare         = _Tr;
    using allocator_type      = _Alloc;
#if _HAS_CXX17
    using node_type = _Node_handle<_List_node<value_type, typename allocator_traits<_Alloc>::void_pointer>, _Alloc,
        _Node_handle_map_base, _Kty, _Ty>;
#endif // _HAS_CXX17

    static constexpr bool _Multi    = _Mfl;
    static constexpr bool _Standard = true;

    template <class... _Args>
    using _In_place_key_extractor = _In_place_key_extract_map<_Kty, _Args...>;

    _Umap_traits() = default;

    explicit _Umap_traits(const _Tr& _Traits) noexcept(is_nothrow_copy_constructible_v<_Tr>) : _Tr(_Traits) {}
}
```

traits里的`_Uhash_compare`定义如下

```cpp
template <class _Kty, class _Hasher, class _Keyeq>
class _Uhash_compare
    : public _Uhash_choose_transparency<_Kty, _Hasher, _Keyeq> { // traits class for unordered containers
public:
    enum { // parameters for hash table
        bucket_size = 1 // 0 < bucket_size
    };

    _Uhash_compare() noexcept(
        conjunction_v<is_nothrow_default_constructible<_Hasher>, is_nothrow_default_constructible<_Keyeq>>)
        : _Mypair(_Zero_then_variadic_args_t{}, _Zero_then_variadic_args_t{}, 0.0f) {}
    }
```

`_Hash`的结构如下

```cpp
template <class _Traits>
class _Hash { // hash table -- list with vector of iterators for quick access
protected:
    using _Mylist             = list<typename _Traits::value_type, typename _Traits::allocator_type>;
    using _Alnode             = typename _Mylist::_Alnode;
    using _Alnode_traits      = typename _Mylist::_Alnode_traits;
    using _Node               = typename _Mylist::_Node;
    using _Nodeptr            = typename _Mylist::_Nodeptr;
    using _Mutable_value_type = typename _Traits::_Mutable_value_type;

    using _Key_compare   = typename _Traits::key_compare;
    using _Value_compare = typename _Traits::value_compare;

public:
    using key_type = typename _Traits::key_type;

    using value_type      = typename _Mylist::value_type;
    using allocator_type  = typename _Mylist::allocator_type;
    using size_type       = typename _Mylist::size_type;
    using difference_type = typename _Mylist::difference_type;
    using pointer         = typename _Mylist::pointer;
    using const_pointer   = typename _Mylist::const_pointer;
    using reference       = value_type&;
    using const_reference = const value_type&;

    using iterator =
        conditional_t<is_same_v<key_type, value_type>, typename _Mylist::const_iterator, typename _Mylist::iterator>;
    using const_iterator = typename _Mylist::const_iterator;

    using _Unchecked_iterator       = conditional_t<is_same_v<key_type, value_type>,
        typename _Mylist::_Unchecked_const_iterator, typename _Mylist::_Unchecked_iterator>;
    using _Unchecked_const_iterator = typename _Mylist::_Unchecked_const_iterator;

    using _Aliter = _Rebind_alloc_t<_Alnode, _Unchecked_iterator>;

    static constexpr size_type _Bucket_size = _Key_compare::bucket_size;
    static constexpr size_type _Min_buckets = 8; // must be a positive power of 2
    static constexpr bool _Multi            = _Traits::_Multi;

    template <class _TraitsT>
    friend bool _Hash_equal(const _Hash<_TraitsT>& _Left, const _Hash<_TraitsT>& _Right);

protected:
    _Hash(const _Key_compare& _Parg, const allocator_type& _Al)
        : _Traitsobj(_Parg), _List(_Al), _Vec(_Al), _Mask(_Min_buckets - 1), _Maxidx(_Min_buckets) {
        // construct empty hash table
        _Max_bucket_size() = _Bucket_size;
        _Vec._Assign_grow(_Min_buckets * 2, _List._Unchecked_end());
#ifdef _ENABLE_STL_INTERNAL_CHECK
        _Stl_internal_check_container_invariants();
#endif // _ENABLE_STL_INTERNAL_CHECK
    }
}
```

每次插入

```cpp
template <class _Keyty, class... _Mappedty>
    pair<_Nodeptr, bool> _Try_emplace(_Keyty&& _Keyval_arg, _Mappedty&&... _Mapval) {
        const auto& _Keyval = _Keyval_arg;
        const auto _Hashval = _Traitsobj(_Keyval);
        auto _Target        = _Find_last(_Keyval, _Hashval);
        if (_Target._Duplicate) {
            return {_Target._Duplicate, false};
        }

        _Check_max_size();
        _List_node_emplace_op2<_Alnode> _Newnode(_List._Getal(), piecewise_construct,
            _STD forward_as_tuple(_STD forward<_Keyty>(_Keyval_arg)),
            _STD forward_as_tuple(_STD forward<_Mappedty>(_Mapval)...));
        if (_Check_rehash_required_1()) {
            _Rehash_for_1();
            _Target = _Find_last(_Traits::_Kfn(_Newnode._Ptr->_Myval), _Hashval);
        }

        return {_Insert_new_node_before(_Hashval, _Target._Insert_before, _Newnode._Release()), true};
}
```

这里看到会检查需不需要rehash，如果需要的话要重新查找节点的位置，最后插入新的节点。可以在`_Uhash_compare`中看到，这个load factor，默认都是1.
