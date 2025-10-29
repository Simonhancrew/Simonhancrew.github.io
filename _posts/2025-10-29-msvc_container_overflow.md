---
title: container overflow in msvc
date: 2025-10-29 21:00:00 +0800
categories: [Blogging, windows, msvc]
tags: [writing]
---

主要是msvc下release-asan模式编译，但是有些静态库是没带asan option编译的，这个在vs2022里默认情况下会触发编译问题

具体的[文档](https://github.com/MicrosoftDocs/cpp-docs/blob/main/docs/sanitizers/error-container-overflow.md)参考

主要原因是vs2022下，开启了annotations，此时为了不违反odr原则，编译器会加点检查的东西在代码里，比如在vs2022下编译就会遇到如下的报错

```
my_static.lib(my_code.obj) : error LNK2038: mismatch detected for 'annotate_vector': value '0' doesn't match value '1' in main.obj
```

根据文档的方案，针对string + vector其实有两个方法可以避免

1. 用macro关闭，比如加上`-D_DISABLE_VECTOR_ANNOTATION`和`-D_DISABLE_STRING_ANNOTATION`
2. 其余的库也要使用asan编译的option

如果是第三方库没有源码的情况下，推荐使用第一种方法
