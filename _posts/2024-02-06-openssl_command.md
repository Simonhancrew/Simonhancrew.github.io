---
layout: post
title:  "openssl command"
category: tools
date:   2024-02-06
---

openssl在命令行层面有一些很方便的命令，比如s_client和x509。这里记录一些常用的命令。


## s_client

s_client命令可以简单的开启一个连接上的客户端，可以用来测试ssl连接。

常用的

```bash
openssl s_client -connect host:port -tls1_3 -servername ""
```

## X509

证书的数据管理

```bash
openssl x509 -in cert.crt -text -noout
```
