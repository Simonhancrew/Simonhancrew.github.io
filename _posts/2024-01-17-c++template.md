---
layout: post
title:  "c++ template"
category: c++
date:   2024-01-17
---

主要说下模板的声明和实现分离


如果不用`std::variant`的话，要实现模板声明和实现分离，需要在头文件中声名，然后在cpp里定义模板，然后要继续在cpp里声明所有要用的类型

```c++
// header.h
template <typename T>
void show(T t);

// header.cpp
template <typename T>
void show(T t) {
  std::cout << t << std::endl;
}

template void show<int>(int);
template void show<std::string>(std::string);
template void show<const char*>(std::vector<int>);
```

另外注意一下`"xxxx"s`的用法, 等价在构造一个string，并且这个string不会被中间的\0截断
