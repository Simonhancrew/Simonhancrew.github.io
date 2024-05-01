---
title: 如何将vscode打造成c++ ide
date: 2023-11-12 14:10:00 +0800
categories: [Blogging, tools]
tags: [writing]
---

推荐安装llvm的工具链，这样就能使用clangd作为LSP，实际跳转非常准确，而且速度也很快。

在vscode里面只要hook一个插件就可以，插件的名字就是clangd

需要注意的是，如果你安装了多个clangd，那么可能要在vscode的clangd插件里指明clangd的路径

## comile_commands.json的生成

首先比较现代的构建系统，基本支持在generation的过程中就能生成compile_commands.json，比如cmake，bazel，ninja等等

但是如果你使用的是makefile，那么就需要借助第三方工具了，比如bear

### cmake如何生成compdb

```bash
cmake -DCMAKE_EXPORT_COMPILE_COMMANDS=on
```

### ninja/gn如何生成compdb

其实我比较推荐的是用gn的时候就生成了, 以下ninja的步骤需要在build目录下已经生成了build.ninja

```bash
ninja -t compdb > compile_commands.json
```

如果是gn的话，会方便不少， gen的时候带上`--add-export-compile-commands=<label_pattern>`, 后面的形如`//build/*`，这样就能生成compile_commands.json了，另外可以直接在.gn里指定导出

```json
export_compile_commands = [
  "//base/*",
  "//tools:doom_melon",
]
```

### makefile导出

makefile稍微复杂点，需要借助bear

```bash
bear make
```

### 使用clang-format和clang-tidy

clang-format其实不用过多介绍，会让你写的代码非常符合预设的code style，而且还能自动格式化，非常方便

clang-tidy可以根据自己的需要，预设一些规则(当然他也依赖compdb), 这个的好处是他会根据mordern c++给你写的代码一些warning和修改建议

配合vscode和clangd插件，显得尤为好用。

### 如何在vscode里配置clangd的fallback option

`Clangd: Fallback Flags`中添加要的选项就可以了

设置完之后可能要重启一下vscode

### 关于find all reference

在配置完compdb之后很有用的一个功能就是find all reference，这玩意在找所有函数在哪里使用的时候很方便。

跳转虽然在compdb之后就能用了，但目前看在虚函数的部分处理的其实不是很好，我不知道这是啥问题

虚函数如果有默认实现的话，他会默认跳虚函数的实现，这时候其实你可能要看子类的实现，这个时候从基类->find all reference然后找到所有的实现切过去就好

### 关闭头文件自动添加

这个在clangd argument里可以自己+一下"--header-insertion=never"

这个看需求了，主要我有时候发现他会+一些很奇怪的头文件，而且是平台相关的，所以在跨平台的时候可能编译不过，就有点烦

### 设置compdb的路径

`--compile-commands-dir=${workspaceFolder}/build`
