---
layout: post
title: "Source Code Reading"
date: 2022-06-07
---

### 源码阅读的方法论

阅读我司的源码部分，开始有些许烦，但是还是在这里总结一下以前读的步骤吧。

1. 工欲善其事必先利其器，开始的过程中一定要搭建出完整的环境，然后在整体构架的基础上了解所有的功能。

2. 明确任务的目标，不要泛泛而读。重视文档，了解整体的构架。

2. 读代码，泛读 + 精读。开始整体把握流程，随后看调用细节。一般只需要关注自己的目标模块就行，不要陷入其他的部分。

3. 利用源码调试技巧和调试工具，一定会有不太理解的部分，这个时候善于利用debugger

4. 可以适当的猜测和魔改源码，看看后续的影响

5. 善用test，配合test做一个调试

然后读的方式主要有两种，第三种其实是融合的方法： 

+ 第一种是Top->down的方法，从整体入口开始读，然后分小模块继续读。这样的优势是比较好的能把握到整体的调用流程，缺点是就是对模块的代码可能把握度不够（主要是底层的部分）

+ 第二种是Bottom->up的方法，从底层的模块开始读起，这种读法的优点就是会非常深刻的看到小组件的功能，但是大构架的串联可能在脑海中的整体性稍差。

+ 第三种是Mixed的方法，其实也算是前两个方法的集大成吧，最终都要走到这一步。

### 调试的一些经验

关于断点:

+ 条件断点：配合入参之类的值，做一个条件断
+ 断点溯源：配合调用栈了解整体的调用流程，一步一步往上看
+ 修改源码：主要我本次需要做一些迁移的任务，所以可以看看原版和迁移之后的区别。
+ log插入法： 打log，看一下参数的变化.适合一直在变的参数，不适合打断点看。

关于多线程:
+ todo
### 阅读完之后应该做的事

1. 可以给自己一个cheatsheet，记录一下阅读的过程和收获

2. 思考一下设计的问题，如果是我的话，应该怎么设计这个模块。然后将我自己的和原框架对比，思考下孰优孰劣

3. 精读部分代码。一定会有那么一部分代码，让你惊为天人或者读完一振，甚至完全读不懂(除了写的垃圾外)，精读这一份源码就可以了。