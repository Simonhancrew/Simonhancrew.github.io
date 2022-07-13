---
layout: post
title: "Google Code Style C++"
date: 2022-07-13
---

# Google Code Style命名规则

为了统一随后的风格，在Google Code Style上对自己的代码书写风格做一个统一的约定。

## 文件名

全部小写，单词之间用下划线隔开。


```cpp
// 形如
fake_server.cc
```

## 类名

首字母大写，不用下划线。

```cpp
class SegTree{

};

using ItemMap = unordered_map<int,Item>;

typedef unordered_map<int,Node> NodeMap;
```

## 普通变量和成员变量

普通变量全部小写，下划线连接。成员变量同理，并用下划线结尾。

```
// 普通变量

string raw_buffer;

// 成员变量

string row_buffer_;

```

## 常量命名

constexpr 和 const 这类在程序运行时始终保持不变的，以k开头，首字母大写。

```
const int kBirthDay = 1300;
```

## 函数命名

统一不用下划线，首字母大写。


```
func MyDaily()
```

## 命名空间

尽量用单个单词，如果有多个的话，全部小写，下划线连接。

## 枚举命名

内部成员名，形如const命名规则。外部使用大写字母开头的方式

## 宏命名

全部大写，单词用下划线分开。

## 行长度和tab，函数定义和声明

不超过80，不使用tab，全部空格，缩进 2 space

返回类型和函数名同行。大括号不换行。

## 预处理命令

从行首开始，不做缩进。








