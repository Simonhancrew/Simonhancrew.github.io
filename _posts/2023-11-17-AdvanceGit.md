---
layout: post
title:  "Advance Git"
category: tools
date:   2023-11-17
---

记一下git的一下操作

### alias

```bash
git config --global alias.st status
git config --global alias.ci commit
git config --global alias.co checkout
```

或者你在`~/.gitconfig`里面直接加上

```bash
[alias]
st=status

```

### 如何看一个commit的行数层面上的改变

```bash

git log --stat <commit-sha>

```

### add指定的文件的几行

```base
git add -p
```

然后根据需要选择s或者e, 当然如果有GUI直接用GUI吧。。。

### blame

```bash
git blame -L 10,20 <file>
```

看一下最近一次的10-20行的改动
