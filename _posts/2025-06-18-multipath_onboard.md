---
title: multipath结束
date: 2025-06-18 22:30:00 +0800
categories: [Blogging, network]
tags: [writing]
---

做了快一年的multipath项目结束了，最后一个月发现了6-7个传输协议的bug，终于勉强达到上线的标准了

### 传输协议

我们其实有自己的传输协议，而且已经天然支持多路径传输。传输协议下的发包组件可以是基于

+ tcp
+ udp

上层协议会serilize成一个个packet，发包组件会将这些packet打包成一个个datagram发送出去。

传输协议大概有基础分层，更上层不必关系，只列跟multipath相关的层

+ connection
+ path

connection可以理解成一个channel，是连接的概念，path是连接的一个路径。一个connection可以有多个path，对应一个path内是一个socket做收发

具体的cc + lost detection + stats 等功能都在path上实现。

### 方案

为了最小化对原有握手的影响，我们的方案设计的也非常tricky

1. 在connection握手阶段，我们会带上connection id(uint64_t)，这个id会在path上被记录下来, 同时带上multipath的flag，告诉对端这是一个multipath连接
2. 在握手结束之后，connection会被创建出来，connection内会有一个init path，这个init path其实是用设备默认路由来收发包的，等价这个path其实没被关联到网卡上
3. 此时connection已经具备收发data packet的能力了，接下来我们会开始新path的握手
4. 新path的握手会调用create path，指定local跟remote的ip和port，但是这个local可以是virtual id，能唯一对应到网卡
5. 此时握手步骤跟connection类似，也会带上connection id，作为connection的key，找到peer的connection，随后在peer的connection中创建path
6. 对端会告诉local，新path创建完成，此时可以close开始的init path（因为没跟网卡做关联）

### 问题

tricky的方案一定是会遇到很多问题的，尤其是传输这种丢包不确定场景 + 包可能乱序的场景下，corner case会非常多。

最大的问题其实是两方面

1. 握手存在延迟
2. connection不稳定，经常收到RST

#### 延迟问题


#### RST问题

