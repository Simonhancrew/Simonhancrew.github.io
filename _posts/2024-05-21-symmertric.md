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

IV初始向量在CBC，CFB模式下，仅影响前16字节(128位加密方式)的块，推荐使用CTR模式.

### 2.0 什么是AES

加密算法分可恢复和不可恢复

可恢复的又分对称加密和非对称加密，AES是对称加密

Advanced Encryption Standard(高级加密标准)，DES之后的继任者，是一种对称加密算法，密钥长度支持128、192、256位，分别对应AES-128、AES-192、AES-256

### 2.1 AES的工作模式

```text
+---------+     +------------+     +-----------+     +------------+     +---------+
|         |     |            |     |           |     |            |     |         |
|  Plain  | --> | AES Encrypt| --> | Transmit  | --> | AES Decrypt| --> |  Plain  |
|  Text   |     |    with Key|     |           |     |    with Key|     |  Text   |
|         |     |            |     |           |     |            |     |         |
+---------+     +------------+     +-----------+     +------------+     +---------+
```

一种比较简单的加解密方式，XOR。

### 2.3 AES的概念

1. Block Size: 128 bits
2. key size: 128/192/256 bits
3. 加密模式: ECB、CBC、CFB、OFB、CTR
4. 初始向量: IV
5. 填充方式: PKCS5Padding、PKCS7Padding、NoPadding
6. 附加消息: AAD

#### 2.3.1 Block Size

首先，**block size**, AES是一种[分组加密](https://zh.wikipedia.org/wiki/%E5%88%86%E7%BB%84%E5%AF%86%E7%A0%81), 每次加密的数据块大小是固定的，AES的数据块大小是128位，也就是16字节。这是AES规定的，不可更改的。

#### 2.3.2 Key Size

其次，**key size**, AES的密钥长度支持128、192、256位，分别对应AES-128、AES-192、AES-256。这是AES的一个特点，也是AES的一个优点，因为密钥长度越长，破解的难度就越大。

| 算法    | 分组长度 | 密钥长度 | 迭代轮数 |
| ------- | -------- | -------- | -------- |
| AES-128 | 128位    | 128      | 10       |
| AES-192 | 128位    | 192      | 12       |
| AES-256 | 128位    | 256位    | 14       |

越往下，越安全

#### 2.3.3 加密模式

第三，**加密模式**，基本能分成5种，看[wiki](https://zh.wikipedia.org/wiki/%E5%88%86%E7%BB%84%E5%AF%86%E7%A0%81%E5%B7%A5%E4%BD%9C%E6%A8%A1%E5%BC%8F)

| 工作模式                    | 描述                                                                                         | 典型应用                                                       |
| --------------------------- | -------------------------------------------------------------------------------------------- | -------------------------------------------------------------- |
| ECB (Electronic Codebook)   | 每个块独立加密，同样的明文块和密钥产生同样的密文块。                                         | 不推荐在需要高安全性的系统中使用，因为它不提供严格的数据混淆。 |
| CBC (Cipher Block Chaining) | 每个明文块与前一个密文块进行异或操作后，再进行加密。                                         | 在需要数据完整性的系统中常用，如HTTPS、SSH。                   |
| CFB (Cipher Feedback)       | 一次处理s位，上一块密文作为加密算法的输入单元，产生的伪随机数输出与明文xor作为下一单元的密文 | 面向数据流的通用传输认证                                       |
| OFB (Output Feedback)       | 跟CFB类似，加密算法的输入是上一次加密的输出，且使用整个分组                                  | 噪声信道上的传输                                               |
| CTR (Counter)               | 每个明文分组都与一个经过加密的计数器相相xor，对每个后续分组，计数器递增                      | 在需要并行处理的系统中常用，如硬盘加密。                       |

#### 2.3.4 初始向量

第四，**初始向量**，IV，Initialization Vector，一般是加密过程初始化产生的随机向量。加密和解密过程需要同一组IV。

假设以CBC为例子的话，在每一个明文块被加密之前，会让明文和密文做xor。IV作为初始化向量，会跟第一个明文块做xor(因为之前没有任何被加密的明文)，后续的每一个明文块和它前一个明文块所加密出的密文块相异或。这样能保证密文块的不重

#### 2.3.5 填充方式

因为密钥往往只能对确定长度的数据进行处理加密，数据的长度是不确定的，因此需要填充，一般是对最后一个块做额外的处理，常用的模式就是PKCS5Padding、PKCS7Padding、NoPadding，这个可以通过openssl的接口, `EVP_CIPHER_CTX_set_padding`来设置

#### 2.3.6 附加消息

AAD，一般不是应用层会关注的数据，一般是包含在协议里的纯数据，需要保护完整性，但不用加密。

## 3 GCM

### 3.1 什么是CTR加密模式

还是分块，但是每次只要知道nonce + counter(一个从0到n的编号) + key就可以了。这个加密模式可以并行执行，但没办法做消息完整性验证。所以得带一个mac（Message Authentication Code）

### 3.2 什么是GCM

GCM = GMAC + CTR, 具体算法的逻辑看下wiki

## REF

### 1. [EVP Symmetric Encryption and Decryption](https://wiki.openssl.org/index.php/EVP_Symmetric_Encryption_and_Decryption)

### 2. [EVP Authenticated Encryption and Decryption](https://wiki.openssl.org/index.php/EVP_Authenticated_Encryption_and_Decryption) 

### 3. [EVP Asymmetric Encryption and Decryption of an Envelope](https://wiki.openssl.org/index.php/EVP_Asymmetric_Encryption_and_Decryption_of_an_Envelope)

### 4. [How to do encryption using AES in Openssl](https://stackoverflow.com/questions/9889492/how-to-do-encryption-using-aes-in-openssl)

### 5. [OpenSSL using EVP vs. algorithm API for symmetric crypto](https://stackoverflow.com/questions/10366950/openssl-using-evp-vs-algorithm-api-for-symmetric-crypto)

### 6. [OpenSSL EVP API](https://wiki.openssl.org/index.php/EVP)

### 7. [什么是 AES-GCM加密算法](https://blog.csdn.net/T0mato_/article/details/53160772)

### 8. [伽罗瓦/计数器模式](https://zh.wikipedia.org/wiki/%E4%BC%BD%E7%BD%97%E7%93%A6/%E8%AE%A1%E6%95%B0%E5%99%A8%E6%A8%A1%E5%BC%8F)

### 9. [分组密码工作模式](https://zh.wikipedia.org/wiki/%E5%88%86%E7%BB%84%E5%AF%86%E7%A0%81%E5%B7%A5%E4%BD%9C%E6%A8%A1%E5%BC%8F)

### 10. [GCM](https://juejin.cn/post/6844904122676690951)

