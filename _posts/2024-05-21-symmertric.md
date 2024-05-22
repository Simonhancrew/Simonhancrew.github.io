---
title: ssl + 对称加密
date: 2024-05-21 12:29:00 +0800
categories: [Blogging, crytpo, ssl]
tags: [writing]
---

聊一下怎么用ssl的接口做对称加密 + 解密。

## 1. AES

看一段[demo](https://github.com/Simonhancrew/recipes/blob/master/crypto/sample/aes_crypto.cpp)

简单的说几个参数

1. IV: Initialization Vector, 一般是加密过程初始化产生的随机向量。加密和解密过程需要同一组IV
2. key, 密钥，加解密共一个，一般是随机生成的，如果不是协商出来 + 随机的话，基本不安全
3. aad, Additional Authenticated Data, 附加认证数据，用于认证加密数据的完整性，加解密是同一个add
4. tag, 用于认证加密数据的完整性，一般是encrypto生成的，要在decrypto的时候提供

剩下的就是必要参数了，比如明文和密文。

因此，如果你自定义了一个加密/解密的流，其实你就定义了一个协议。

另外尽量不要使用`AES_*`的参数，能用`EVP_*`的话就尽量使用后者，因为在内部实现中，会有很多高级指令对运算操作加速

## 2. iv + key的生成

## REF

### 1. [EVP Symmetric Encryption and Decryption](https://wiki.openssl.org/index.php/EVP_Symmetric_Encryption_and_Decryption)

### 2. [EVP Authenticated Encryption and Decryption](https://wiki.openssl.org/index.php/EVP_Authenticated_Encryption_and_Decryption) 

### 3. [EVP Asymmetric Encryption and Decryption of an Envelope](https://wiki.openssl.org/index.php/EVP_Asymmetric_Encryption_and_Decryption_of_an_Envelope)

### 4. [How to do encryption using AES in Openssl](https://stackoverflow.com/questions/9889492/how-to-do-encryption-using-aes-in-openssl)

### 5. [OpenSSL using EVP vs. algorithm API for symmetric crypto](https://stackoverflow.com/questions/10366950/openssl-using-evp-vs-algorithm-api-for-symmetric-crypto)

### 6. [OpenSSL EVP API](https://wiki.openssl.org/index.php/EVP)

