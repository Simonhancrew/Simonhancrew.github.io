---
title: leveldb在win下编译
date: 2024-11-10 17:01:10 +0800
categories: [Blogging, leveldb, compile]
tags: [writing]
---

leveldb在unix下编译很简单，不赘述。

之前看代码都是在linux下编译的，在win下看相关的实现的话，还是要配合编译一下的

本次编译环境

+ win == 11
+ cmake == 3.31.0
+ ninja == 1.12.1
+ clang == 18.1.0rc
+ msvc == 2022

到手clone文件之后首先还是把third_party里的submodule先更新一下，然后就是编译了，

```shell
git submodule init
git submodule update
```

leveldb的编译组织全靠cmake的，如果你不想用vs看代码的话，就需要生成compdb，cmake下compdb目前只支持make + ninja作为generator的情况下生成。

所以我选择ninja做为generator，不用开vs，直接vscode + clangd插件看代码

```shell
cmake -B build -DCMAKE_EXPORT_COMPILE_COMMANDS=ON -DCMAKE_BUILD_TYPE=Debug -GNinja
```

然后直接走编译流程就行了，但我这里碰到部分warn as error的问题，主要在benchmark里，所以在编译bench这个lib的时候需要额外的加上部分cflgas

这部分要+在leveldb里, 主要是windows下sscanf会让你用更安全的ms扩展版本。

```cmake
if (WIN32)
  target_compile_options(leveldb
    PUBLIC
      -Wno-deprecated-declarations)
endif(WIN32)
```

另外一部分要加在benchmark里

```cmake
if (WIN32)
  target_compile_options(benchmark 
    PUBLIC 
  -Wno-invalid-offsetof -Wno-shorten-64-to-32)
endif(WIN32)
```


然后熟悉clangd的那套的话，自己陪一下就可以看代码了。
