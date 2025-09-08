---
title: leveldb arena
date: 2025-09-08 16:46:00 +0800
categories: [Blogging, leveldb]
tags: [writing]
---

leveldb里的组件，负责内存分配的，写的比较简陋

接口就两个，一个分配普通内存，一个分配对齐内存的

```cpp
class Arena {
 public:
  // Return a pointer to a newly allocated memory block of "bytes" bytes.
  char* Allocate(size_t bytes);

  // Allocate memory with the normal alignment guarantees provided by malloc.
  char* AllocateAligned(size_t bytes);
 private:
  // Allocation state
  char* alloc_ptr_;
  size_t alloc_bytes_remaining_;

  // Array of new[] allocated memory blocks
  std::vector<char*> blocks_;

  // Total memory usage of the arena.
  //
  // TODO(costan): This member is accessed via atomics, but the others are
  //               accessed without any locking. Is this OK?
  std::atomic<size_t> memory_usage_;
};
```

具体实现，分别看了，先看简单点的普通分配

```cpp
inline char* Arena::Allocate(size_t bytes) {
  // The semantics of what to return are a bit messy if we allow
  // 0-byte allocations, so we disallow them here (we don't need
  // them for our internal use).
  assert(bytes > 0);
  if (bytes <= alloc_bytes_remaining_) {
    char* result = alloc_ptr_;
    alloc_ptr_ += bytes;
    alloc_bytes_remaining_ -= bytes;
    return result;
  }
  return AllocateFallback(bytes);
}

char* Arena::AllocateFallback(size_t bytes) {
  if (bytes > kBlockSize / 4) {
    // Object is more than a quarter of our block size.  Allocate it separately
    // to avoid wasting too much space in leftover bytes.
    char* result = AllocateNewBlock(bytes);
    return result;
  }

  // We waste the remaining space in the current block.
  alloc_ptr_ = AllocateNewBlock(kBlockSize);
  alloc_bytes_remaining_ = kBlockSize;

  char* result = alloc_ptr_;
  alloc_ptr_ += bytes;
  alloc_bytes_remaining_ -= bytes;
  return result;
}

char* Arena::AllocateNewBlock(size_t block_bytes) {
  char* result = new char[block_bytes];
  blocks_.push_back(result);
  memory_usage_.fetch_add(block_bytes + sizeof(char*),
                          std::memory_order_relaxed);
  return result;
}
```

简单讲就是有剩下空间分，就优先用剩下的，不然就fallback到开新的block

AllocateFallback的逻辑里，如果要的空间大于1 / 4的默认blocksize(4k),就直接分出去，不然就可以走分一部分 + 剩下的继续下次分的逻辑了

比较搞的是这里只有memory_usage_是个atomic，别的都是普通值，感觉线程管理语义有点割裂

然后就是对齐分的

```cpp
char* Arena::AllocateAligned(size_t bytes) {
  const int align = (sizeof(void*) > 8) ? sizeof(void*) : 8;
  static_assert((align & (align - 1)) == 0,
                "Pointer size should be a power of 2");
  size_t current_mod = reinterpret_cast<uintptr_t>(alloc_ptr_) & (align - 1);
  size_t slop = (current_mod == 0 ? 0 : align - current_mod);
  size_t needed = bytes + slop;
  char* result;
  if (needed <= alloc_bytes_remaining_) {
    result = alloc_ptr_ + slop;
    alloc_ptr_ += needed;
    alloc_bytes_remaining_ -= needed;
  } else {
    // AllocateFallback always returned aligned memory
    result = AllocateFallback(bytes);
  }
  assert((reinterpret_cast<uintptr_t>(result) & (align - 1)) == 0);
  return result;
}
```

这里主要的逻辑在前几句, 

```cpp
const int align = (sizeof(void*) > 8) ? sizeof(void*) : 8; // 确定对齐要求，默认8，不然就用指针大小
static_assert((align & (align - 1)) == 0,
            "Pointer size should be a power of 2"); // 确认是2的幂
size_t current_mod = reinterpret_cast<uintptr_t>(alloc_ptr_) & (align - 1); // 这里计算位偏移，等价在alloc_ptr % align
size_t slop = (current_mod == 0 ? 0 : align - current_mod); // 需要跳过的字节，因为需要当前指针式位置是对齐的(%align为0)
```

剩下的逻辑就是类似的了

可以看到的是这里只有分的逻辑，没有回收的部分，整体的内存析构在arena析构的时候一把梭了

```cpp
Arena::~Arena() {
  for (size_t i = 0; i < blocks_.size(); i++) {
    delete[] blocks_[i];
  }
}
```
