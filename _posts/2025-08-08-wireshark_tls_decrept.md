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

openssl之类的其实支持往SSLKEYLOGFILE写入，只需要在建立tls连接的时候调用接口

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
