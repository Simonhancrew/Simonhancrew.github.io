---
layout: post
title: "Knuth-Shuffle的概率推导"
date: 2021-09-28
---

正常DFS然后random pick的话，经历一个全排列，是一个O(n!)的解法，基本不可能使用。

这里依托概率问题能够生成一个伪随机打乱的数列，可以看做完美洗牌。

```
void KnuthShuffle(vector<int> &nums){
    int n = nums.size();
    random_device r;
    std::mt19937 gen(r);
    for(int i = n-1;i>=0;i--){
        uniform_int_distribution<int> uniform(0,i);
        swap(nums[i],nums[uniform(gen)]);
    }
}
```

比较容易想的是最后一个，从当前所有的位置中选一个和最后一个位置的数交换（最后一个位置也是可选的，可以自己交换自己），任意一个数放到这个位置的概率是1/n

之前的位置就有了先决性，我们倒数第二个数是从第一次挑剩下的数中选择的

$$
\frac{n - 1}{n}
$$
且选择的概率是 

$$
\frac{1}{n -1}
$$
结果就是
$$
\frac{n - 1}{n} \cdot \frac{1}{n - 1} =  \frac{1}{n}
$$
比较容易理解的场景就是一个扫雷游戏，开始在1-n的位置生成地雷，之后random_shuffle一次之后，把这些地雷分散到棋盘不同的格子中

