---
title: clash meta android编译
date: 2025-11-27 1:20:00 +0800
categories: [Blogging, android]
tags: [writing]
---

主要是自己编一下clash meta for android，记录一下成功过程

### 1 clone repo

```bash
git clone git@github.com:MetaCubeX/ClashMetaForAndroid.git
```

### 2. submodule

里面有mihomo的submodule

```bash
git submodule update --init --recursive
```

### 3. android sdk path

默认已经安装好了

1. jdk11
2. android sdk
3. CMake
4. Golang

在win下编译出来的，jdk不能装android studio自带的，会报错

```
Error occurred during initialization of VM
Option -XX:+UseZGC not supported
```

JDK 不支持 ZGC（Z Garbage Collector）


### 4. 改app id

这一步可选，主要为了换皮逃系统检测, 在根目录下创建`local.properties`

```
# config your ownn applicationId, or it will be 'com.github.metacubex.clash'
custom.application.id=com.my.compile.clash
# remove application id suffix, or the applicaion id will be 'com.github.metacubex.clash.alpha'
remove.suffix=true
```

### 5. 创建keystore

```bash
keytool -v -genkey -alias testalias -keyalg EC -groupname secp256r1 -validity 36500 -keystore release.keystore
```

把repo里默认的覆盖掉，我是直接删掉了

然后就配置一下`local.properties`

```
keystore.path=${user.home}/ClashMetaForAndroid/release.keystore
keystore.password=123456
key.alias=test
key.password=123456
```

### 6. 编译

官方的readme是直接gradlw编译的

```
./gradlew app:assembleAlphaRelease
```

### REF

1. [ClashMetaForAndroid](https://github.com/MetaCubeX/ClashMetaForAndroid)
