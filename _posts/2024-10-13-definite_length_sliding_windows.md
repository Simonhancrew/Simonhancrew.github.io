---
title: 定长滑动窗口问题
date: 2024-10-13 19:01:10 +0800
categories: [Blogging, algorithm, sliding windows]
tags: [writing]
---

定长滑动窗口问题其实有一个大的for模板

```cpp
for (int i = 0;i < n;++i) {
  // 1. 进入窗口
  if (i < k - 1) continue;
  // 2. 更新答案
  // 3. 离开窗口
}
```

可以做题试试就知道了

[大小为 K 且平均值大于等于阈值的子数组数目 1317](https://leetcode.cn/problems/number-of-sub-arrays-of-size-k-and-average-greater-than-or-equal-to-threshold/description/)

这部分中，有一个很经典的问题可以转换为k长度的滑动窗口，比如头尾取数问题，求最大点数和

这个可以转化为长度为k的中间窗口最小问题

题目是[1423. 可获得的最大点数](https://leetcode.cn/problems/maximum-points-you-can-obtain-from-cards/description/)

还有一类问题，比如[2653. 滑动子数组的美丽值](https://leetcode.cn/problems/sliding-subarray-beauty/description/), 这类问题在找窗口里的nth element，这种一般数据值范围都比较小，可以用计数排序来解决，如果数据范围比较大的话，就要试试二分查找了。
