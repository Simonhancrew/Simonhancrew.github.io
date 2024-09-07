---
title: std::exchange
date: 2024-09-07 14:40:00 +0800
categories: [Blogging, cpp, c++]
tags: [writing]
---

`std::exchange`是c++14提供的一个函数模板，在`<utility>`里

```cpp
template<class T, class U = T>
constexpr // since C++20
T exchange(T& obj, U&& new_value)
    noexcept( // since C++23
        std::is_nothrow_move_constructible<T>::value &&
        std::is_nothrow_assignable<T&, U>::value
    )
{
    T old_value = std::move(obj);
    obj = std::forward<U>(new_value);
    return old_value;
}
```

设置新的值，返回旧的值，所以配合这个实现，你得这样写

```cpp
int new_val = 0;
int target = 42;
new_val = std::exchange(target, new_val);
```

## 使用场景1，move

```cpp
struct S {
  int n{42}; // default member initializer
  S() = default;

  S(S &&other) noexcept : n{std::exchange(other.n, 0)} {}

  S &operator=(S &&other) noexcept {
    // safe for this == &other
    n = std::exchange(other.n, 0); // move n, while leaving zero in other.n
    return *this;
  }
};

int main() {
  S s;
  // s = s; // 1. Error! does not match the move assigment operator
  s = std::move(s); // 2. OK! explicitly move

  std::cout << s.n << "\n"; // Outputs: 42
}
```

## 使用场景2，输出helper

```cpp
int main() {
  std::vector<int> vec(10);
  std::ranges::iota(vec, 0);

  std::cout << "[";
  const char *delim = "";
  for (auto val : vec) {
    std::cout << std::exchange(delim, ", ") << val;
  }

  // Outputs: [0, 1, 2, 3, 4, 5, 6, 7, 8, 9]
  std::cout << "]\n";
}
```

这种新旧值需要交换得，可以考虑使用`std::exchange`

## 使用场景3, 所有权交换

```cpp
void transfer_ownership(auto &obj) {
  [o = std::move(obj)] {}();
}

// NOTE: okay
std::vector<int> v1(10);
std::ranges::iota(v1, 0);
transfer_ownership(v1);
// Outputs: []
fmt::print("v1: {}\n", v1);

// NOTE: possiblely still has value
std::optional<int> foo{42};
transfer_ownership(foo);
// true
fmt::print("has_value: {}\n", foo.has_value());
```

使用`std::exchange`可以代码更少

```cpp
void transfer_ownership(auto& obj) {
    [o = std::exchange(obj, {})] {}();
}

std::optional<int> foo{ 42 };
transfer_ownership(foo);
// false
fmt::print("has_value: {}\n", foo.has_value());
```

## REF

1. [std::exchange use cases](https://www.cppmore.com/2023/07/25/stdexchange-use-cases/)
