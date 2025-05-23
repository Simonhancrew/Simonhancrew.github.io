---
title: false sharing和mesi
date: 2025-02-11 21:28:00 +0800
categories: [Blogging, os, thread]
tags: [writing]
---

主要看到两个文章，之前看brpc的mesi相关的线程极致优化，当时没看全

MESI协议 是一种用于维护多核CPU缓存一致性的协议，确保多个核心访问共享内存时数据的一致性, 它是缓存一致性协议中最常见的一种

+ M (Modified)：缓存行已被修改（与主内存不一致），且只有当前核心拥有最新数据。

+ E (Exclusive)：缓存行与主内存一致，且只有当前核心拥有该缓存行。

+ S (Shared)：缓存行与主内存一致，但可能被多个核心共享。

+ I (Invalid)：缓存行无效（不能使用），需要从主内存或其他核心重新加载数据。

### 1 行为

MESI协议通过状态转换来维护缓存一致性

#### 1.1 读取

如果缓存行状态为`M/E/S`，核心可以直接读取数据。

如果状态为`I`，核心需要从主内存或其他核心加载数据，并将状态更新为`S`或 `E`。

#### 1.2 写入

+ 如果缓存行状态为`M`，核心可以直接写入。
+ 如果状态为`E`，核心将状态改为`M`并写入。

+ 如果状态为`S`，核心需要广播Invalidate信号，使其他核心的缓存行失效（变为`I`），然后将状态改为`M`并写入。

+ 如果状态为 I，核心需要从主内存或其他核心加载数据，然后执行写入操作。

#### 1.3 状态转换

+ `M` → `S`：当其他核心请求共享该缓存行时，当前核心将数据写回主内存，状态改为 S。
+ `E` → `S`：当其他核心请求共享该缓存行时，状态改为`S`
+ `S` → `I`：当其他核心修改该缓存行时，当前核心的缓存行失效。
+ `I` → `E/S`：当核心从主内存或其他核心加载数据时，状态改为`E`或`S`。
+ `E` → `M`：当核心写入数据时，状态改为`M`。
+ `S` (→ `I` )→ `M`: 核心需要广播 Invalidate 信号, 然后将状态改为 M 并写入

### 2 false sharing

主要存在一种情况，两个不同线程，访问的不同数据，在一个cache line上，导致了false sharing

假设这个数据的cache line是`X`

1. 内存地址`X`的数据为10, Core1和Core2的缓存行状态均为`I`。
2. Core1读取`X`, Core1从主内存加载数据，缓存行状态变为`E`。
3. Core2读取`X`, Core2从主内存加载数据，Core1和Core2的缓存行状态均变为`S`
4. Core1修改`X`, Core1发送`Invalidate`信号，Core2的缓存行状态变为 I。Core1的缓存行状态变为`M`，数据更新为20。
5. Core2 读取 `X`, Core2发现缓存行状态为`I`，向 Core1请求数据, Core1 将数据写回主内存，状态变为`S`, Core2加载数据，状态变为`S`

| step       | Core1 | Core2 |
| ---------- | ----- | ----- |
| 初始       | I     | I     |
| core1读取x | E     | I     |
| core2读取x | S     | S     |
| core1写入x | M     | I     |
| core2读取x | S     | S     |
