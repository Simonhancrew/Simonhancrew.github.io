---
title: 如何看编译产物的target cpu
date: 2024-09-26 10:01:10 +0800
categories: [Blogging, compile]
tags: [writing]
---

### objdump

```
objdump -f [lib.a/lib.so]
```

这个能直接看到打包产物的elf format

### ar + file

另外的方法就是ar + file

```
ar x lib.a
file *.o
```
