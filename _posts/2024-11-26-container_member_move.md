---
title: vector扩容时候的move问题
date: 2024-11-26 22:25:10 +0800
categories: [Blogging, move, vector, lambda, c++]
tags: [writing]
---

先抛出问题代码，下面代码的这种情况下存在一个内部指针泄露问题，伴随vector扩容，ItemManager的function内部存储的指针会指向错误的Item对象

因此，下面的这个Item类，虽然是move constructable的，但是实际上因为使用了lambda + capture了this指针，等价于将内部的this外传，item manager没有随之处理这种move的情况。

```cpp
#include <functional>
#include <iostream>
#include <map>
#include <vector>

using namespace std;

struct ItemManager {
  void Register(int id, std::function<void()> &&func) {
    funcs_[id] = std::move(func);
  }
  void Remove(int id) { funcs_.erase(id); }
  std::map<int, std::function<void()>> funcs_;
};

ItemManager &GetItemManager() {
  static ItemManager item_manager;
  return item_manager;
}

struct Item {
  int id;
  Item(int id) : id(id) {
    cout << "create item" << id << '\n';
    GetItemManager().Register(
        id, [this, id]() { printf("item %d, this=%p\n", id, this); });
  }
  ~Item() {
    cout << "destroy item" << id << '\n';
  }
  Item(Item &&item) : id(item.id) { cout << "move item" << id << '\n'; }
  void print() const { printf("item %d, this=%p\n", id, this); }
};

int main() {
  std::vector<Item> item_list;
  auto print = [&item_list]() {
    cout << "item_list print\n";
    for (const auto &it : item_list) {
      it.print();
    }
    cout << "function print\n";
    for (const auto &[k, v] : GetItemManager().funcs_) {
      v();
    }
    cout << '\n';
  };
  item_list.emplace_back(1);
  print();
  item_list.emplace_back(2);
  print();
  item_list.emplace_back(3);
  print();
}
```

单独考虑扩容时候的情况，此时遇到cap的limit，会先开辟一片新的空间.

之前的老vec中的item会挪移到新开辟的vec中去，此时在新vec空间内的item，内部的this指针有了新的指向，外部manager内function所捕获的this指针没有更新，因此会指向错误的item对象。

下面的代码是vector扩容时候的emplace操作

```cpp
template <class... _Valty>
_CONSTEXPR20 pointer _Emplace_reallocate(const pointer _Whereptr, _Valty&&... _Val) {
    // reallocate and insert by perfectly forwarding _Val at _Whereptr
    _Alty& _Al        = _Getal();
    auto& _My_data    = _Mypair._Myval2;
    pointer& _Myfirst = _My_data._Myfirst;
    pointer& _Mylast  = _My_data._Mylast;

    _STL_INTERNAL_CHECK(_Mylast == _My_data._Myend); // check that we have no unused capacity

    const auto _Whereoff = static_cast<size_type>(_Whereptr - _Myfirst);
    const auto _Oldsize  = static_cast<size_type>(_Mylast - _Myfirst);

    if (_Oldsize == max_size()) {
        _Xlength();
    }

    const size_type _Newsize = _Oldsize + 1;
    size_type _Newcapacity   = _Calculate_growth(_Newsize);

    const pointer _Newvec           = _STD _Allocate_at_least_helper(_Al, _Newcapacity);
    const pointer _Constructed_last = _Newvec + _Whereoff + 1;
    pointer _Constructed_first      = _Constructed_last;

    _TRY_BEGIN
    _Alty_traits::construct(_Al, _STD _Unfancy(_Newvec + _Whereoff), _STD forward<_Valty>(_Val)...);
    _Constructed_first = _Newvec + _Whereoff;

    if (_Whereptr == _Mylast) { // at back, provide strong guarantee
        if constexpr (is_nothrow_move_constructible_v<_Ty> || !is_copy_constructible_v<_Ty>) {
            _STD _Uninitialized_move(_Myfirst, _Mylast, _Newvec, _Al);
        } else {
            _STD _Uninitialized_copy(_Myfirst, _Mylast, _Newvec, _Al);
        }
    } else { // provide basic guarantee
        _STD _Uninitialized_move(_Myfirst, _Whereptr, _Newvec, _Al);
        _Constructed_first = _Newvec;
        _STD _Uninitialized_move(_Whereptr, _Mylast, _Newvec + _Whereoff + 1, _Al);
    }
    _CATCH_ALL
    _STD _Destroy_range(_Constructed_first, _Constructed_last, _Al);
    _Al.deallocate(_Newvec, _Newcapacity);
    _RERAISE;
    _CATCH_END

    _Change_array(_Newvec, _Newsize, _Newcapacity);
    return _Newvec + _Whereoff;
}
```

伴随_Newvec被分配，这里唯一的考虑_Uninitialized_move的情况，其余类似，内存会被腾挪到新开辟的array里

```cpp
template <class _InIt, class _Alloc>
_CONSTEXPR20 _Alloc_ptr_t<_Alloc> _Uninitialized_move(
    const _InIt _First, const _InIt _Last, _Alloc_ptr_t<_Alloc> _Dest, _Alloc& _Al) {
    // move [_First, _Last) to raw _Dest, using _Al
    // note: only called internally from elsewhere in the STL
    using _Ptrval     = typename _Alloc::value_type*;
    auto _UFirst      = _STD _Get_unwrapped(_First);
    const auto _ULast = _STD _Get_unwrapped(_Last);
    if constexpr (conjunction_v<bool_constant<_Iter_move_cat<decltype(_UFirst), _Ptrval>::_Bitcopy_constructible>,
                      _Uses_default_construct<_Alloc, _Ptrval, decltype(_STD move(*_UFirst))>>) {
#if _HAS_CXX20
        if (!_STD is_constant_evaluated())
#endif // _HAS_CXX20
        {
            _STD _Copy_memmove(_UFirst, _ULast, _STD _Unfancy(_Dest));
            return _Dest + (_ULast - _UFirst);
        }
    }

    _Uninitialized_backout_al<_Alloc> _Backout{_Dest, _Al};
    for (; _UFirst != _ULast; ++_UFirst) {
        _Backout._Emplace_back(_STD move(*_UFirst));
    }

    return _Backout._Release();
}
```

最后，move完成了，摧毁老的array

```cpp
_CONSTEXPR20 void _Change_array(const pointer _Newvec, const size_type _Newsize, const size_type _Newcapacity) {
    // orphan all iterators, discard old array, acquire new array
    auto& _Al         = _Getal();
    auto& _My_data    = _Mypair._Myval2;
    pointer& _Myfirst = _My_data._Myfirst;
    pointer& _Mylast  = _My_data._Mylast;
    pointer& _Myend   = _My_data._Myend;

    _My_data._Orphan_all();

    if (_Myfirst) { // destroy and deallocate old array
        _STD _Destroy_range(_Myfirst, _Mylast, _Al);
        _ASAN_VECTOR_REMOVE;
        _Al.deallocate(_Myfirst, static_cast<size_type>(_Myend - _Myfirst));
    }

    _Myfirst = _Newvec;
    _Mylast  = _Newvec + _Newsize;
    _Myend   = _Newvec + _Newcapacity;
    _ASAN_VECTOR_CREATE;
}
```

因此，任意对象，只要存在内部资源泄露(指针或非POD对象)问题，且自定义的move函数没有对过期资源的更新操作，这个对象其实是不能放进内存连续的顺序容器的

更求稳，能放进容器的对象，尽量是POD的。

删除头部元素，后面元素的move + 摧毁旧的item的操作同理，相似

### 判断类是不是可移动的

```cpp
template <typename T> 
void check_if_moveable() {
  std::cout << "Is move constructible: " << std::boolalpha
            << std::is_move_constructible<T>::value << std::endl;
  std::cout << "Is move assignable: " << std::is_move_assignable<T>::value
            << std::endl;
}
```

c++的标准在不同编译器 + 版本上一直在变，这个实际上要自己确认一下具体对象的情况，不过这个函数可以作为一个参考
