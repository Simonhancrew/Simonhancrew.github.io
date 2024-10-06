---
title: 入侵式链表
date: 2024-10-06 15:01:10 +0800
categories: [Blogging, intrusive link list, 入侵式链表, c++, data struct]
tags: [writing]
---

简单看下跟常规链表不一样的入侵式链表，和相对而言，这种链表的优势

### 使用`std::list<T>`

看一个erase的场景

```cpp
std::list<T> list;

void erase_node(auto* ptr) {
  for (auto it = list.begin(); it != list.end(); ++it) {
    if (&*it == ptr) {
      list.erase(it);
      break;
    }
  }
}
```

这种场景下，链表的删除显得非常的愚蠢，随机访问的效果很拉跨

### 使用入侵式链表

入侵链表要求link字段直接侵入被管理连接的对象中，侵入式链表让它存在于被链接的结构中。

比如

```cpp
template <typename T>
struct IntrusiveListNode {
  IntrusiveListNode* prev;
  IntrusiveListNode* next;
  T* owner;
};

struct UserData {
  // ...
  InstruiveListNode list;
};
```

具体的对比可以看看[boost的描述](https://www.boost.org/doc/libs/1_79_0/doc/html/intrusive/intrusive_vs_nontrusive.html)

### 优势

这里的优势一般都针对`std::list<T*>`来说的, 这种场景下入侵链表的优势会比较大

比如你要实现LRU cache，需要把`entry`同时加入`HashMap`和`List`中，这时候如果用intrusive list的话，其实拿到entry就能操作链表了

1. Data Locality，不用解引用（std::list<T*>），直接访问
2. 脱离容器的生命周期，侵入式链表需要用户自己管理数据节点生命周期
3. 对于侵入式链表，我们拿到数据就可以将这个节点从链表中摘除，而无需再去定位其 iterator，然后再去找到对应的容器 erase 这个 iterator。

### 实现

基本都是很简单的双向循环链表

#### linux里的实现

+ [linux里的实现](https://github.com/torvalds/linux/blob/v5.18/include/linux/list.h)

+ [侵入式链表结构的定义](https://github.com/torvalds/linux/blob/v5.18/include/linux/types.h)

```c
struct list_head {
  struct list_head *next, *prev;
};
```

使用方法见

```c
// https://github.com/torvalds/linux/blob/v5.18/kernel/sched/sched.h#L376
struct task_group {
  // 省略一些业务逻辑
  struct list_head list;
};

// https://github.com/torvalds/linux/blob/v5.18/kernel/sched/core.c

/*
 * Default task group.
 * Every task in system belongs to this group at bootup.
 */
struct task_group root_task_group;
LIST_HEAD(task_groups);

list_add(&root_task_group.list, &task_groups);
```

c里数据结构比较简单，都能按照约定做layout，所以用几个宏就能拿到结构体的起始位置

```c
// https://github.com/torvalds/linux/blob/v5.18/include/linux/list.h#L519

/**
 * list_entry - get the struct for this entry
 * @ptr:        the &struct list_head pointer.
 * @type:       the type of the struct this is embedded in.
 * @member:     the name of the list_head within the struct.
 */
#define list_entry(ptr, type, member) \
        container_of(ptr, type, member)

// https://github.com/torvalds/linux/blob/v5.18/include/linux/container_of.h#L17

/**
 * container_of - cast a member of a structure out to the containing structure
 * @ptr:          the pointer to the member.
 * @type:         the type of the container struct this is embedded in.
 * @member:       the name of the member within the struct.
 *
 */
#define container_of(ptr, type, member) ({                            \
        void *__mptr = (void *)(ptr);                                 \
        static_assert(__same_type(*(ptr), ((type *)0)->member) ||     \
                      __same_type(*(ptr), void),                      \
                      "pointer type mismatch in container_of()");     \
        ((type *)(__mptr - offsetof(type, member))); })
```

当然，c++就别用这个offset做事了，原因是[c++的虚函数表，多继承等等，会让这个offset不对](https://stackoverflow.com/questions/1129894/why-cant-you-use-offsetof-on-non-pod-structures-in-c)

#### c++里怎么办

看看[doom3的实现](https://github.com/Edgarins29/Doom3/blob/d80c4d8341a35a8d9bc71324dcfc2be583295c03/neo/idlib/containers/LinkList.h)

```cpp
template< class type >
class idLinkList {
 public:
  // 省略

 private:
  idLinkList* head;
  idLinkList* next;
  idLinkList* prev;
  type*       owner;
};
```

用的时候类似这样

```cpp
class idEntity {
  idLinkList<idEntity> spawnNode;  // for being linked into spawnedEntities list
};

idEntity::idEntity() {
  spawnNode.SetOwner( this );
}

/*
================
idLinkList<type>::InsertBefore

Places the node before the existing node in the list.  If the existing node is the head,
then the new node is placed at the end of the list.
================
*/
template< class type >
void idLinkList<type>::InsertBefore(idLinkList &node) {
  Remove();

  next       = &node;
  prev       = node.prev;
  node.prev  = this;
  prev->next = this;
  head       = node.head;
}

/*
================
idLinkList<type>::Remove

Removes node from list
================
*/
template< class type >
void idLinkList<type>::Remove( void ) {
  prev->next = next;
  next->prev = prev;

  next = this;
  prev = this;
  head = this;
}

/*
================
idLinkList<type>::AddToEnd

Adds node at the end of the list
================
*/
template< class type >
void idLinkList<type>::AddToEnd(idLinkList &node) {
  InsertBefore(*node.head);
}

ent->spawnNode.AddToEnd(spawnedEntities);
```

实际上，拿到owner就可以拿到具体的结构体了

### REF

1. [Unrolled linked list](https://www.wikiwand.com/en/articles/Unrolled_linked_list)
2. [C++ 侵入式链表总结](https://zhuanlan.zhihu.com/p/524894979)
