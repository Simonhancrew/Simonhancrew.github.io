---
title: openssl command"
date: 2024-02-06 14:10:00 +0800
categories: [Blogging, tools]
tags: [writing]
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

## REF 

1. [rfc-8446](https://www.rfc-editor.org/rfc/rfc8446)
2. [tls shake procedure](https://tls13.xargs.org/#client-handshake-keys-calc)
