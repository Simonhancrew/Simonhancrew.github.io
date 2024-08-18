---
title: order of evaluation
date: 2024-08-18 11:10:00 +0800
categories: [Blogging, c++, evaluation]
tags: [writing]
---

In every simple assignment expression E1 = E2 and every compound assignment expression E1 @= E2, every value computation and side effect of E2 is sequenced before every value computation and side effect of E1.

这个是在c++17之后才生效的，gcc在std=c++14的时候就可以复现一个问题

```cpp
#include <iostream>
#include <vector>

using namespace std;

int cnt = 0;

struct node {
  int w;
};
vector<node> tree;

int update(int k) {
  int rt = ++cnt;
  node newnode;
  newnode.w = 0;
  tree.push_back(newnode);
  if (k > 0) {
    //int temp = update(k/2);
    //tree[rt].w = temp;
    tree[rt].w = update(k / 2);
  }
  return 8;
}

int main() {
  tree.push_back({0});
  update(2);
  int n = tree.size();
  for (int i = 1; i < tree.size(); i++) {
    cout << tree[i].w << endl;
  }
  return 0;
}
```

这个问题在于tree[rt]可能提前计算，但是在右侧的update中包含对tree的push back操作，如果造成了tree的扩容，那么此时的tree[rt]其实是悬垂的。

目前在gcc13.2能成功复现，加上`--fsanitize=address`可以复现报错信息，但是clang13.0.1就无法复现了

## REF

1. [order of evaluation](https://en.cppreference.com/w/cpp/language/eval_order#Rules)
