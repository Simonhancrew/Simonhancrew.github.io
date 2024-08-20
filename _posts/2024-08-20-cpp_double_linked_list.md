---
title: 构造一个c++双向链表数据结构
date: 2024-08-20 20:20:00 +0800
categories: [Blogging, c++, data-structure]
tags: [writing]
---

主要是针对[old-new-thing-Constructing nodes of a hand-made linked list, how hard can it be?](https://devblogs.microsoft.com/oldnewthing/20240819-00/?p=110145&ocid=oldnewthing_eml_tnp_autoid307_title)

最简单的就是

```cpp
struct node
{
    node* prev;
    node* next;
};
```

更进一步的思考就是天然让他做一个环形链表

```cpp
struct node
{
    node* prev = this;
    node* next = this;
};
```

如果还想继续搞一个构造，从一个已经存在的节点构造一个新的back insert节点，要注意的是，这里不会拿引用，或者const&, 这是为了避免错误的搞成了拷贝构造

```cpp
struct node
{
    node* prev = this;
    node* next = this;

    node() = default;

    // Construct a node after a specific node
    node(node* other) :                      
        prev(other),                         
        next(other->next)                    
    {                                        
        prev->next = this;                   
        next->prev = this;                   
    }                                        
};
```

然后有了back insert，你也应该有before insert

```cpp
struct node
{
    node* prev = this;
    node* next = this;

    node() = default;

    // Construct a node after a specific node
    node(node* other) :
        prev(other),
        next(other->next)
    {
        prev->next = this;
        next->prev = this;
    }

    // Construct a node before a specific node
    node(node* other) :                       
        prev(other->prev),                    
        next(other)                           
    {                                         
        prev->next = this;                    
        next->prev = this;                    
    }                                         
};
```

但这样就重名了，然后正常人的解决方案可能是factory func或者tag dispatch

```cpp
struct node
{
    node* prev = this;
    node* next = this;

    node() = default;

    static node create_after(node* other) 
    {                                     
        node n{ other->prev, other };     
        n.prev->next = &n;                
        n.next->prev = &n;                
        return n;                         
    }                                     
                                          
    static node create_before(node* other)
    {                                     
        node n{ other, other->next };     
        n.prev->next = &n;                
        n.next->prev = &n;                
        return n;                         
    }                                     
};

// usage
node a;
node b = node::create_after(&a); // a->b
node c = node::create_before(&b); // a->c->b
```

nrvo在这里可能不work，盲猜

1. 函数内部使用了返回对象的地址
2. 要靠编译器

所以怎么说都不一定能很好的work，所以他又搞了tag dispatch的版本

```cpp
struct place_after_t {};                       
inline constexpr place_after_t place_after{};  
                                               
struct place_before_t {};                      
inline constexpr place_before_t place_before{};

struct node
{
    node* prev = this;
    node* next = this;

    node() = default;

    // Construct a node after a specific node
    node(place_after_t, node* other) :
        prev(other),
        next(other->next)
    {
        prev->next = this;
        next->prev = this;
    }

    // Construct a node before a specific node
    node(place_before_t, node* other) :
        prev(other->prev),
        next(other)
    {
        prev->next = this;
        next->prev = this;
    }

    static node create_after(node* other) 
    {                                     
        return { place_after, other };    
    }                                     
                                          
    static node create_before(node* other)
    {                                     
        return { place_before, other };   
    }                                     
};

// Sample usage 1: Using tags
node a;
node b(place_after, &a); // list is a->b
node c(place_before, &b); // list is  a->c->b

// Sample usage 2: Using factories
node a;
node b = node::create_after(&a); // list is a->b
node c = node::create_before(&b); // list is  a->c->b
```

这个看起来也很怪，这玩意没道理是static的，node都有instance了才有理由去调用，不如直接搞成成员方法.

```cpp
struct node
{
    node* prev = this;
    node* next = this;

    node() = default;

    node create_after() { return { place_after, this }; }
    node create_before() { return { place_before, this }; }

private:
    struct place_after_t {};
    inline constexpr place_after_t place_after{};

    struct place_before_t {};
    inline constexpr place_before_t place_before{};

    node(place_after_t, node* other) :
        prev(other),
        next(other->next)
    {
        prev->next = this;
        next->prev = this;
    }

    node(place_before_t, node* other) :
        prev(other->prev),
        next(other)
    {
        prev->next = this;
        next->prev = this;
    }
};

// Sample usage
node a;
node b = a.create_after(); // list is a->b
node c = b.create_before(); // list is a->c->b
```

实质上，整个走下来，还是需要点想法
