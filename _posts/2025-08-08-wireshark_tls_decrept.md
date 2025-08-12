---
title: wireshake解密tls数据包
date: 2025-08-08 00:08:14 +0800
categories: [Blogging, wireshake]
tags: [writing]
---

wireshark里天然支持tls解密，比较简单的方法是设置SSLKEYLOGFILE环境变量，wireshark会自动读取这个文件。


### 创建SSLKEYLOGFILE文件

windows

```
setx SSLKEYLOGFILE C:\Path\To\sslkeylogfile.txt
```

unix 

```
export SSLKEYLOGFILE=/Path/To/sslkeylogfile.txt
```


### 设置wireshark

"Edit" > "Preferences" and look for "Protocols" > "TLS".

specify the (Pre)-Master-Secret log filename to the file you created above.

### 自定义代码写入

openssl之类的其实支持往SSLKEYLOGFILE写入，只需要在建立tls连接的时候调用接口, 这里你也可以就往日志里写，因为wireshark其实是支持指定key log file的。

```
static FILE *g_keylog_file = nullptr;

static void KeyLogCallback(const SSL *ssl, const char *line) {
  fprintf(g_keylog_file, "%s\n", line);
  fflush(g_keylog_file);
}

void SetUpSSL() {
  // init ssl ctx
  const char *keylog_file = getenv("SSLKEYLOGFILE");
  if (keylog_file) {
    g_keylog_file = fopen(keylog_file, "a");
    if (g_keylog_file == nullptr) {
      perror("fopen");
      return false;
    }
    SSL_CTX_set_keylog_callback(ctx.get(), KeyLogCallback);
  }
  // other SSL setup code...
}
```

### 关于格式

一般没有更多密钥推到的话，你可能就会看到一个

+ CLIENT_RANDOM: 48 bytes for the master secret, encoded as 96 hexadecimal characters (for SSL 3.0, TLS 1.0, 1.1 and 1.2)

tls1.3还需要

+ CLIENT_EARLY_TRAFFIC_SECRET: the hex-encoded early traffic secret for the client side (for TLS 1.3)
+ CLIENT_HANDSHAKE_TRAFFIC_SECRET: the hex-encoded handshake traffic secret for the client side (for TLS 1.3)
+ SERVER_HANDSHAKE_TRAFFIC_SECRET: the hex-encoded handshake traffic secret for the server side (for TLS 1.3)
+ CLIENT_TRAFFIC_SECRET_0: the first hex-encoded application traffic secret for the client side (for TLS 1.3)
+ SERVER_TRAFFIC_SECRET_0: the first hex-encoded application traffic secret for the server side (for TLS 1.3)
+ EARLY_EXPORTER_SECRET: the hex-encoded early exporter secret (for TLS 1.3, used for 0-RTT keys in older QUIC drafts).
+ EXPORTER_SECRET: the hex-encoded exporter secret (for TLS 1.3, used for 1-RTT keys in older QUIC drafts)

### REF

+ [The SSLKEYLOGFILE Format for TLS](https://www.ietf.org/archive/id/draft-thomson-tls-keylogfile-00.html)
