---
title: sparse/dense array
date: 2025-02-11 22:36:00 +0800
categories: [Blogging, c++]
tags: [writing]
---

[entt](https://github.com/skypjack/entt)的[介绍](https://docs.google.com/presentation/d/1PbCH2IRg8lW08JlUz-xQEQjTo_Fr5n9QZOTEjGqZnuU/edit#slide=id.g1e93a5b7c98_0_172)



有个用局部性原理优化的, 优势是

+ Fastest unordered iteration speed
+ Constant insertion and deletion
+ Constant lookup by ID
+ ID (but not pointer) stability
+ OK (but not amazing) deterministic iteration speed
+ Avoid with non trivially relocatable types

代码大概长这个样子

```cpp
#include <cassert>
#include <vector>


template <typename T>
class DenseSparseArray {
public:
  void insert(uint32_t entity, const T &value) {
    if (entity >= sparse.size()) {
      sparse.resize(entity + 1, -1); // -1 表示无效索引
    }

    if (sparse[entity] == -1) {
      sparse[entity] = dense.size();
      dense.push_back({entity, value});
    }
  }

  void erase(uint32_t entity) {
    if (contains(entity)) {
      // 0
      size_t dense_idx = sparse[entity];
      auto &last = dense.back();
      auto last_entity = last.entity;

      std::swap(dense[dense_idx], last);
      sparse[last_entity] = dense_idx;

      dense.pop_back();
      sparse[entity] = -1;
    }
  }

  // 访问元素
  T &operator[](uint32_t entity) {
    assert(contains(entity));
    cout << "entity idx: " << sparse[entity] << endl;
    cout << "desze size: " << dense.size() << endl;
    return dense[sparse[entity]].value;
  }

  // 判断是否存在
  bool contains(uint32_t entity) const {
    return entity < sparse.size() && sparse[entity] != -1;
  }

  // 迭代器支持
  auto begin() { return dense.begin(); }
  auto end() { return dense.end(); }

private:
  struct Element {
    uint32_t entity;
    T value;
  };

  std::vector<int> sparse;    // 稀疏数组（存储索引）
  std::vector<Element> dense; // 密集数组（实际数据）
};

int main() {

  DenseSparseArray<int> arr;

  arr.insert(100, 42); // 插入 entity=100
  arr.insert(200, 77); // 插入 entity=200

  std::cout << arr[100]; // 输出 42

  arr.erase(100); // 删除 entity=100

  cout << arr[200] << '\n';
  cout << arr[100] << '\n'; // assert 触发
}
```
