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

### log

```bash
git log -n：显示最近的 n 个提交。例如，git log -5 将显示最近的 5 个提交。
git log --since=2.weeks：显示最近两周内的提交。
git log --author="author"：显示特定作者的提交。例如，git log --author="John" 将显示 John 的所有提交。
git log --grep="message"：搜索提交信息中包含特定文本的提交。例如，git log --grep="bug fix" 将搜索提交信息中包含 "bug fix" 的提交。
git log --stat：显示每个提交的文件修改统计信息
git log --graph：以 ASCII 图形显示分支和合并历史。
git log commit1..commit2：显示两个提交之间的差异。例如，git log HEAD..origin/master 将显示当前 HEAD 和 origin/master 之间的差异。(单commit也可以)
git log -- file：显示特定文件的历史。例如，git log -- README.md 将显示 README.md 文件的修改历史。

```

### proxy

```base
git config --global http.proxy http://
git config --global https.proxy https://

# 关闭
git config --global --unset http.proxy
git config --global --unset https.proxy
```

另外ssh代理的话稍微复杂一点，需要编译.ssh里的config

```bash
Host github.com
    User git
    ProxyCommand nc -X connect -x local_ip:port %h %p
```

### 全局ignore

```bash
git config --global core.excludesfile ~/.gitignore_global
```

### 设置vim编辑commit信息

```bash
git config --global core.editor vim
```

### 带一下date的信息

```bash
git commit --date="yyyy-mm-ddTHH:MM:SS" -m "Commit message"
```
