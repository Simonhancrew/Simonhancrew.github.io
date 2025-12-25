---
title: xcode出现could not locate device问题的解决
date: 2025-12-25 15:30:00 +0800
categories: [Blogging, apple, xcode, device]
tags: [writing]
---

xcode在使用过程中，有时候会出现could not locate device的问题，导致无法在真机上运行和调试应用。

这里有两个解决办法，第一个就是升级xcode了，但是另外一个改造更小的办法是找到缺失的deveices support然后放到对应的目录下就可以了

1. 检查手机的系统version
2. 在[这个repo](https://github.com/iGhibli/iOS-DeviceSupport)下载对应的version support
3. 将下载的文件放在xcode_path/Contents/Developer/Platforms/iPhoneOS.platform/DeviceSupport目录下

完成之后，重启xcode就可以解决问题了
