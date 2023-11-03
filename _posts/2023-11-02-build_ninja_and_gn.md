---
layout: post
title:  "build ninja and gn"
category: compile
date:   2023-11-02
---

1. 安装vs

2. ninja就用官方的，python3 + ./configure.py --bootstrap, 然后用ninja -t compdb生成一个编译数据库，需要注意的是要打开vs的power shell
    > 需要注意的是，在windows下，clangd的compile db要求是utf-8且只能是LF

3. gn的编译更简单，同样打开vs环境powershell，然后python3 build/gen.py --ide=vs2019 out/vs2019 --args="is_debug=false is_component_build=false"，这里的参数可以参考gn的文档，然后就可以用vs打开out/vs2019/gn.sln了。当然你也可以什么都不加，然后用ninja -C out直接编译出gn来

### 注意事项

1. 要使用ninja的话，需要打开语言设置->非Unicode程序语言->更改系统区域设置->beta版本勾选

2. compile_commands.json的生成需要在vs的pwsh下进行，否则会出现编码问题


### 一些使用gn的资料

1. [chromium gn](https://chromium.googlesource.com/chromium/src/tools/gn/+/48062805e19b4697c5fbd926dc649c78b6aaa138/docs/language.md)

2. [gn quick start](https://gn.googlesource.com/gn/+/main/docs/quick_start.md)

3. [gn cross compile](https://gn.googlesource.com/gn/+/main/docs/cross_compiles.md)

4. [introduce to gn](https://www.topcoder.com/thrive/articles/Introduction%20to%20Build%20Tools%20GN%20&%20Ninja)

5. [using gn](https://docs.google.com/presentation/d/15Zwb53JcncHfEwHpnG_PoIbbzQ3GQi_cpujYwbpcbZo/htmlpresent)
