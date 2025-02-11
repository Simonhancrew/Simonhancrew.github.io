---
title: windows base64使用
date: 2025-02-11 21:28:00 +0800
categories: [Blogging, base64, windows]
tags: [writing]
---

windows下怎么使用base64编码

ms下其实有原生的支持, [doc](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/certutil)

如果你要简单的用的话, option的话，用的时候再查。。。

encode

```shell
certutil [options] -encode infile outfile
```

decode

```shell
certutil [options] -decode infile outfile
```
