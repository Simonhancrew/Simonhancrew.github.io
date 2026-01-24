---
title: ssh Keystroke timing obfuscation
date: 2026-01-24 17:44:00 +0800
categories: [Blogging, ssh]
tags: [writing]
---

是看到了这个[Why does SSH send 100 packets per keystroke?](https://eieio.games/blog/ssh-sends-100-packets-per-keystroke/)

作者在用ssh开发游戏，即使不发送游戏数据（他错误的发了非游戏数据），服务器CPU使用率仍高达50%（理论应该降低100%），用tcpdump分析发现：单个按键操作会触发约270个网络包，其中179个是36字节的小包（这个他是用claude clode帮忙分析pcap的）

然后claude分析发现了下面这坨

```
Packet Size Distribution (413,703 total packets):
274,907 packets (66%): Exactly 36 bytes
138,778 packets (34%): 0 bytes (TCP ACKs)


Further analysis on a smaller pcap pointed to these mysterious packets arriving ~20ms apart.
```

随后他在ssh -vvv的时候看到了obfuscate_keystroke_timing的信息，实锤了问题

根因是在这一句

```
In 2023, ssh added keystroke timing obfuscation. The idea is that the speed at which you type different letters betrays some information about which letters you’re typing. So ssh sends lots of “chaff” packets along with your keystrokes to make it hard for an attacker to determine when you’re actually entering keys.
```

SSH的击键时序混淆功能（2023年新增），为了保护隐私，SSH会发送大量“干扰包”掩盖用户真实的击键节奏，每个按键后会发送49-101个虚假PING包，间隔约20ms


解决方案是

1. 客户端禁用（不理想）：ObscureKeystrokeTiming=no, 因为不想客户端用户都去改配置
2. 服务端修复：fork Go的crypto库，移除对SSH ping扩展的支持
   + 服务器不再声明支持ping@openssh.com扩展
   + 客户端就不会发送干扰包

整体其实是一个比较好的用AI解决实际问题的案例
