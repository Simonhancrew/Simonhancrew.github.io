---
title: leveldb skiplist
date: 2025-12-10 21:20:00 +0800
categories: [Blogging, leveldb]
tags: [writing]
---

下面的介绍基本都是从OI-Wiki上摘录的。

跳表 (Skip List) 是由 William Pugh 发明的一种查找数据结构，支持对数据的快速查找，插入和删除。

跳表的期望空间复杂度为 O(n)，插入和删除操作的期望时间复杂度都为 O(log n)，最坏情况下的时间复杂度为 O(n)。

### 基本思想

基本思想是对有序链表的一个改进，正常来讲，一个链表的查找操作就是从头到尾遍历一遍，时间复杂度为 O(n)。

skiplist在这个基础上引入了分层索引的概念。首先明确几个概念就好了

1. 跳表是有多层的，每一层都是一个有序链表。其中最底层(第0层)的链表包含了所有的元素，其他层的链表则是对底层链表的一个索引。
2. 每个位于i层的元素，有p的概率会出现在i+1层。

所以，在一个n个节点的跳表中，期望包含1/p个元素的层为L(n)层，L(n) = log_(1/p)(n)

在跳表中查找，就是从第L(n)层开始，水平地逐个比较直至当前节点的下一个节点大于等于目标节点，然后移动至下一层。重复这个过程直至到达第一层且无法继续进行操作。此时，若下一个节点是目标节点，则成功查找；反之，则元素不存在。这样一来，查找的过程中会跳过一些没有必要的比较，所以相比于有序链表的查询，跳表的查询更快

### 跳表的实现

#### 构造

```cpp
template <typename Key, class Comparator>
SkipList<Key, Comparator>::SkipList(Comparator cmp, Arena* arena)
    : compare_(cmp),
      arena_(arena),
      head_(NewNode(0 /* any key will do */, kMaxHeight)),
      max_height_(1),
      rnd_(0xdeadbeef) {
  for (int i = 0; i < kMaxHeight; i++) {
    head_->SetNext(i, nullptr);
  }
}
```

这里的 `head_` 是跳表的头节点，`max_height_` 是当前跳表的最大高度，`rnd_` 是一个随机数生成器。

其中头的每一层的next都指向nullptr。

#### 获取节点的最大层数

```cpp
template <typename Key, class Comparator>
int SkipList<Key, Comparator>::RandomHeight() {
  // Increase height with probability 1 in kBranching
  static const unsigned int kBranching = 4;
  int height = 1;
  while (height < kMaxHeight && rnd_.OneIn(kBranching)) {
    height++;
  }
  assert(height > 0);
  assert(height <= kMaxHeight);
  return height;
}
```

这里的 `kBranching` 是一个常量，表示每次增加高度的概率。`kMaxHeight` 是跳表允许的最大高度，这个值是12。

#### 查询

```cpp
template <typename Key, class Comparator>
bool SkipList<Key, Comparator>::Contains(const Key& key) const {
  Node* x = FindGreaterOrEqual(key, nullptr);
  if (x != nullptr && Equal(key, x->key)) {
    return true;
  } else {
    return false;
  }
}

template <typename Key, class Comparator>
typename SkipList<Key, Comparator>::Node*
SkipList<Key, Comparator>::FindGreaterOrEqual(const Key& key,
                                              Node** prev) const {
  Node* x = head_;
  int level = GetMaxHeight() - 1;
  while (true) {
    Node* next = x->Next(level);
    if (KeyIsAfterNode(key, next)) {
      // Keep searching in this list
      x = next;
    } else {
      if (prev != nullptr) prev[level] = x;
      if (level == 0) {
        return next;
      } else {
        // Switch to next list
        level--;
      }
    }
  }
}

template <typename Key, class Comparator>
bool SkipList<Key, Comparator>::KeyIsAfterNode(const Key& key, Node* n) const {
  // null n is considered infinite
  return (n != nullptr) && (compare_(n->key, key) < 0);
}
```

这里prev一直是nullptr，所以prev[level] = x; 这行代码不会被执行。

核心的函数在FindGreaterOrEqual，从最高层的节点开始，如果当查找的key要大于当前节点的key，在当前层继续查找。

否则就切换到下一层继续查找。直到找到一个节点的key大于等于要查找的key，此时这个代码里的next其实已经到了最底层的节点。

#### 插入

```cpp
template <typename Key, class Comparator>
void SkipList<Key, Comparator>::Insert(const Key& key) {
  // TODO(opt): We can use a barrier-free variant of FindGreaterOrEqual()
  // here since Insert() is externally synchronized.
  Node* prev[kMaxHeight];
  Node* x = FindGreaterOrEqual(key, prev);

  // Our data structure does not allow duplicate insertion
  assert(x == nullptr || !Equal(key, x->key));

  int height = RandomHeight();
  if (height > GetMaxHeight()) {
    for (int i = GetMaxHeight(); i < height; i++) {
      prev[i] = head_;
    }
    // It is ok to mutate max_height_ without any synchronization
    // with concurrent readers.  A concurrent reader that observes
    // the new value of max_height_ will see either the old value of
    // new level pointers from head_ (nullptr), or a new value set in
    // the loop below.  In the former case the reader will
    // immediately drop to the next level since nullptr sorts after all
    // keys.  In the latter case the reader will use the new node.
    max_height_.store(height, std::memory_order_relaxed);
  }

  x = NewNode(key, height);
  for (int i = 0; i < height; i++) {
    // NoBarrier_SetNext() suffices since we will add a barrier when
    // we publish a pointer to "x" in prev[i].
    x->NoBarrier_SetNext(i, prev[i]->NoBarrier_Next(i));
    prev[i]->SetNext(i, x);
  }
}

template <typename Key, class Comparator>
typename SkipList<Key, Comparator>::Node* SkipList<Key, Comparator>::NewNode(
    const Key& key, int height) {
  char* const node_memory = arena_->AllocateAligned(
      sizeof(Node) + sizeof(std::atomic<Node*>) * (height - 1));
  return new (node_memory) Node(key);
}
```

这里的插入操作首先会调用 `FindGreaterOrEqual` 来找到要每层插入的位置之前位置，然后生成一个新的节点。


这里的node的实现是

```cpp
// Implementation details follow
template <typename Key, class Comparator>
struct SkipList<Key, Comparator>::Node {
  explicit Node(const Key& k) : key(k) {}

  Key const key;

  // Accessors/mutators for links.  Wrapped in methods so we can
  // add the appropriate barriers as necessary.
  Node* Next(int n) {
    assert(n >= 0);
    // Use an 'acquire load' so that we observe a fully initialized
    // version of the returned Node.
    return next_[n].load(std::memory_order_acquire);
  }
  void SetNext(int n, Node* x) {
    assert(n >= 0);
    // Use a 'release store' so that anybody who reads through this
    // pointer observes a fully initialized version of the inserted node.
    next_[n].store(x, std::memory_order_release);
  }

  // No-barrier variants that can be safely used in a few locations.
  Node* NoBarrier_Next(int n) {
    assert(n >= 0);
    return next_[n].load(std::memory_order_relaxed);
  }
  void NoBarrier_SetNext(int n, Node* x) {
    assert(n >= 0);
    next_[n].store(x, std::memory_order_relaxed);
  }

 private:
  // Array of length equal to the node height.  next_[0] is lowest level link.
  std::atomic<Node*> next_[1];
};
```

里面的next_[1]有点类似c的柔性数组，NewNode分配的内存是 `sizeof(Node) + sizeof(std::atomic<Node*>) * (height - 1)`，所以next_[1]的长度是动态的。

可以看到的是这里考虑了无锁的实现。

#### 删除

这里没有提供显示的删除api，删除的逻辑其实就是执行一边查询的逻辑，中途需要记录要删除的节点是在拿一些的节点后面

最后再执行删除，每一层最后一个key小于删除key的节点，就是需要修改的节点

然后删除可能导致当前最大层数是减小的，需要调整max_height_的值。

这里只需要改造下面的接口，带上Node** prev就可以。

```cpp
// Return the latest node with a key < key.
// Return head_ if there is no such node.
Node* FindLessThan(const Key& key) const;
```

