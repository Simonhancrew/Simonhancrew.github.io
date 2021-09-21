---
layout: post
title: "烽火科技的笔试编程题"
data: 2021-09-21
---

恐怖故事，烽火通信应该算2线企业了吧，但是我看了今天笔试的题目，好像并不简单

第一题打卡就不多说了，第二题就开始背包模型了，数据范围是40，不能爆搜或者二进制枚举。这里我没写实际的题目，思路就是一个背包问题，如果专门针对训练过可能很简单，但是我觉得大多数应届的不知道怎么针对训练dp问题。状态转移直接写成前i个数中，凑成值为k的个数有多少。这里我写过一个类似的，可以参考一下，可以优化成一维。

```c++
int dp[target + 1];
memset(dp,0,sizeof dp);
dp[0] = 1;
for(int i = 1;i <= n;i++){
	for(int j = target;j >= nums[i - 1];j--){
		dp[j] += dp[j - nums[i - 1]];
	}
}
```

第三题是一个编辑距离，又是dp，虽然比较烂大街，但是实际上处理完输入输出也有50行了，一年前没打竞赛的时候看这个代码当时也是裂开。

```c++
const int N = 20,M = 1e3 + 10;

int dp[N][N];
char str[M][N];

int edit_dis(char a[],char b[]){
    int la = strlen(a + 1),lb = strlen(b + 1);
    
    for(int i = 0;i <= la;i++) dp[i][0] = i;
    for(int i = 0;i <= lb;i++) dp[0][i] = i;
    
    for(int i = 1;i <= la;i++){
        for(int j = 1;j <= lb;j++){
            dp[i][j] = min(dp[i - 1][j] + 1,dp[i][j - 1] + 1);
            if(a[i] == b[j]) dp[i][j] = min(dp[i][j],dp[i - 1][j - 1]);
            else dp[i][j] = min(dp[i][j],dp[i - 1][j - 1] + 1);
        }
    }
    return dp[la][lb];
}
```

