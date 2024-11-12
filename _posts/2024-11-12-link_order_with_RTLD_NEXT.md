---
title: dynamic link order with RTLD_NEXT
date: 2024-11-12 20:40:10 +0800
categories: [Blogging, compile, link]
tags: [writing]
---


如果hook了库函数，集成的时候对于运行时库的link顺序其实是有点要求的


+ [计算机系统篇之链接（16）：真正理解 RTLD_NEXT 的作用](https://csstormq.github.io/blog/%E8%AE%A1%E7%AE%97%E6%9C%BA%E7%B3%BB%E7%BB%9F%E7%AF%87%E4%B9%8B%E9%93%BE%E6%8E%A5%EF%BC%8816%EF%BC%89%EF%BC%9A%E7%9C%9F%E6%AD%A3%E7%90%86%E8%A7%A3%20RTLD_NEXT%20%E7%9A%84%E4%BD%9C%E7%94%A8)
+ [Dangers of using dlsym() with RTLD_NEXT](https://optumsoft.com/dangers-of-using-dlsym-with-rtld_next/)
+ [linux hook机制研究](https://zhuanlan.zhihu.com/p/44132805)
