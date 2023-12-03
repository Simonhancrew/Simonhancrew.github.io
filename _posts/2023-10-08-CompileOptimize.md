---
layout: post
title:  "Compile Optimize"
category: compile
date:   2023-10-08
---

编译加速上的一些经验和分析方法，主要针对c++。

实际上在了解完make和ninja之后，发现要加速单translation unit的编译，应该最小化他要做的事，例如include what u want，使用pimpl，使用forward declaration等等。

## 一些compile optimize的经验

1. [c++项目编译优化](https://bot-man-jl.github.io/articles/?post=2022/Cpp-Project-Compile-Optimization)

2. [C++服务编译耗时优化原理及实践](https://tech.meituan.com/2020/12/10/apache-kylin-practice-in-meituan.html)

3. [chrome do and dont](https://github.com/chromium/chromium/blob/master/styleguide/c%2B%2B/c%2B%2B-dos-and-donts.md#minimize-code-in-headers)

4. [constructor destructor behavior](https://www.chromium.org/developers/coding-style/chromium-style-checker-errors/#constructordestructor-errors)
    
    其实我觉得在.h里写=default可能是trival的

5. [Compilation Firewalls-1](https://herbsutter.com/gotw/_100/)

6. [Compilation Firewalls-2](https://herbsutter.com/gotw/_101/)

7. [gg code style suggests](https://google.github.io/styleguide/cppguide.html#The__define_Guard)

8. [chrome componet build](https://github.com/chromium/chromium/blob/master/docs/component_build.md)

9. [webkit build suggestions](https://trac.webkit.org/wiki/AnalyzingBuildPerformance)

10. [clang-time-trace practice](https://www.snsystems.com/technology/tech-blog/clang-time-trace-feature)

11. [excutable-size-analysis](https://snsystems.com/technology/tech-blog/analyzing-the-size-of-the-compiler-executable)
    > 这个是上一个作者的另一篇文章，我感觉挺有意思，和编译相关，包体积分析优化

12. [flame-time-trace](https://aras-p.info/blog/2019/01/16/time-trace-timeline-flame-chart-profiler-for-Clang/)

13. [time-report by clang](https://aras-p.info/blog/2019/01/12/Investigating-compile-times-and-Clang-ftime-report/)

14. [clang build analysis](https://aras-p.info/blog/2019/09/28/Clang-Build-Analyzer/)

目前看来就是尽量最小化include，尽量使用pimpl, 虽然现在的gg code style不再推荐这种方式了，但是我估计他们编译都上集群了，也不再乎这点东西，另外一些缺点其实注意命名和命名空间的话，其实比较少遇到。坑的文章在[这里](https://www.zhihu.com/question/63201378)

## 如何分析编译耗时

编译耗时的分析，可以使用`-ftime-trace`，这个选项会在编译的时候生成一个json文件，然后使用`llvm-profdata`和`llvm-cov`来分析这个json文件，具体可以参考[这里](https://clang.llvm.org/docs/CommandGuide/clang.html#cmdoption-ftime-trace)。

如果使用ninja的话，看编译依赖关系可以使用`ninja -t deps`, 具体看下[文档](https://ninja-build.org/manual.html#_extra_tools)就行

gn看依赖关系的话，可以使用`gn desc`，具体看下[文档](https://gn.googlesource.com/gn/+/master/docs/reference.md#cmd_desc)
