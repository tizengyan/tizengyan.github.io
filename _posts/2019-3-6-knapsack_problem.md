---
layout: post
title:  "背包问题总结（上）"
#date:   2019-03-06 12:12:54
categories: 基础算法
tags: dynamic_programming
excerpt: 总结一下多种条件下的背包问题
author: Tizeng
---

* content
{:toc}

首先记住解决动态规划的三个基本要素：

1. 最优子结构

2. 边界条件

3. 状态转移方程

## 背包问题

背包问题是一个非常典型的考察动态规划应用的题目，对其加上不同的限制和条件，可以衍生出诸多变种，若要全面理解动态规划，就必须对背包问题了如指掌。

题目描述：

一个小偷面前有一堆（n个）财宝，每个财宝有重量`w`和价值`v`两种属性，而他的背包只能携带一定重量的财宝（Capacity），在已知所有财宝的重量和价值的情况下，如何选取财宝，可以最大限度的利用当前的背包容量，取得最大价值的财宝（或求出能够获取财宝价值的最大值）。

### 0-1背包问题

即限定每个物品要么拿（1个）要么不拿（0个）。

乍一看这个问题有两个维度，一个是当前物品`i`，另一个是**当前**容量`c`，于是我们可以用`f[n,C]`来表示将`n`个物品放入容量为`C`的背包可以得到的最大收益，而第`i`个物品无非拿与不拿两种情况，因此可以表示为：

> f[i][c] = max( f[i - 1][c], f[i - 1][c - w[i]] + v[i] )

这便是我们的最优子结构，即不拿第 i 件物品和拿第 i 件物品中的最大值，当然，这里要保证`w[i] <= c`，否则`f[i][c] = f[i - 1][c]`。

```c++
int knapsack(vector<int> v, vector<int> w, int n, int C){
    vector<vector<int> > f(n, vector<int>(C + 1));
    for(int i = 0; i < n; i++)
        f[i][0] = 0;
    for(int j = 1; j <= C; j++)
        if(j >= w[0])
            f[0][j] = v[0];
        else
            f[0][j] = 0;
    for(int i = 1; i < n; i++){
        for(int j = 1; j <= C; j++){
            if(j < w[i]){
                f[i][j] = f[i - 1][j]
            }
            else{
                f[i][j] = max(f[i - 1][j], f[i - 1][j - w[i]] + v[i]);
            }
        }
    }
    return f[n - 1][C];
}
```

此时的时间和空间复杂度都是O(nC)，我们可以对空间复杂度做进一步的优化。如下图所示，我们从第一行开始，从左往右开始填表，可以发现除了第一行外，每一行都只和它的上一行有关（观察状态转移方程亦可知），因此不需要把整个表都存起来，只需要保存两行，这样空间复杂就变成了O(C)。

![dp_table1.jpg](https://github.com/tizengyan/images/raw/master/dp_table1.jpg)

但是这样仍不是最优，我们还可以从右往左开始填表：

```c++
int knapsack(vector<int> v, vector<int> w, int n, int C){
    vector<int> f(C + 1);

    for(int i = 0; i < n; i++){
        for(int j = C; j >= w[i]; j--)
            f[j] = max(f[j], f[j - w[i]] + v[i]);
    }
    return f[C];
}
```

这样相当于永远只保存一行数据，根据前面数组前面的数据更新后面的，最后就得到了上面图片的最后一行。

有时题目会要求我们输出最优解，而不只是最优解的答案，这时我们就无法在空间上对算法进行优化了，因为我们需要每一次变化中保存的值，以回溯最优解：

```c++
int i = n - 1;
int j = C;
while(i >= 0){
    if(f[i][j] == f[i - 1][j])
        cout << "未选第 " << i << " 件物品 << endl;
    else if(f[i][j] == f[i - 1][j - w[i]] + v[i]) {
        cout << "选l 第 " << i << " 件物品 << endl;
        j -= w[i];
    }
    i--;
}
```

这里是要求背包装有最大价值的物品，没有规定必须将背包装满，如果规定背包必须装满，那么除了`f[0]`初始化为0，其他的`f[1~C]`都要初始化为`INT_MIN`，可以理解为没有物品时，如果背包容量为0，那么什么都不装就是刚好装满，价值为0，而如果背包容量大于0，说明初始情况除了`f[0]`外我们哪种情况都装不满，因此把那些无解的情况初始化为负无穷。

#### 问题变种：[LeetCode 416](https://leetcode.com/problems/partition-equal-subset-sum/)

* Given a non-empty array containing only positive integers, find if the array can be partitioned into two subsets such that the sum of elements in both subsets is equal.

输入一组数组，判断是否可以将这个数组一分为二，让分过后的两个子数组中的元素和相等。

这一题和0-1背包问题比较类似，我们可以尝试用背包问题的思路去解决。本质上题目需要我们从数组中选取若干元素，看看存不存在一种组合使之的和为所有元素之和的二分之一（这个可以看做背包容量）。

首先排除和为奇数的情况，因为如果和为奇数则一定不可平分。接下来分析最优子结构，我们将问题分为两个维度，所能选取的物品（数组中的数字）数量和背包的大小（所有和的二分之一），那么对于第 i 件物品我有拿或不拿两种选择，如果不拿，这时的结论完全取决于我们对第 i - 1 件物品的结论，如果拿，首先要满足这件物品的值nums[i]不超过 j ，那么此时的结论就取决于背包大小为`j - nums[i]`时的结论，记`dp[i][j]`表示我能拿到前`i - 1`件物品时（下标从0开始），能否取出和为`j`的组合的结论，那么状态转移方程就可以写为`dp[i][j] = dp[i - 1][j] || dp[i - 1][j - nums[i]]`，这里数组`dp`为`bool`型。最后是边界条件，如果`j = 0`，则什么都不取就可以达到目标，因此对于任何 i ，`dp[i][0]`都初始化为`true`，然后是`i = 0`的情况，即只有第一个物品，那么只有当`nums[0] == j`的时候才符合条件，且要满足`nums[0] <= sum/2`。

由以上思路就可以写出下面的代码：

```c++
bool canPartition(vector<int>& nums) {
    if(nums.size() == 1)
        return false;
    int sum = 0;
    for(const auto& n : nums)
        sum += n;
    if(sum & 1)
        return false;
    vector<vector<bool> > dp(nums.size(), vector<bool>(sum/2 + 1, false)); // 这里要注意不要忘了加一
    for(int i = 0; i < nums.size(); i++)
        dp[i][0] = true;
    if(nums[0] <= sum/2)
        dp[0][nums[0]] = true;
    for(int i = 1; i < nums.size(); i++){
        for(int j = 1; j <= sum/2; j++){
            if(j >= nums[i])
                dp[i][j] = dp[i - 1][j] || dp[i - 1][j - nums[i]];
            else
                dp[i][j] = dp[i - 1][j];
        }
    }
    return dp[nums.size() - 1][sum/2];
}
```

这个思路的时间复杂和空间复杂度都是O(nC)，我们可以像之前的例子一样将其优化，只用一个一维数组就解决问题：

```c++
bool canPartition(vector<int>& nums) {
    if(nums.size() == 1)
        return false;
    int sum = 0;
    for(const auto& n : nums)
        sum += n;
    if(sum & 1)
        return false;
    int half = sum / 2;
    vector<bool> dp(half + 1, false);
    dp[0] = true;
    for(int i = 1; i < nums.size(); i++){
        for(int j = half; j >= nums[i]; j--){
            dp[j] = dp[j] || dp[j - nums[i]];
        }
    }
    return dp[half];
}
```

这里第一行需要单独初始化。

### 完全（无界）背包问题

如果不限定每种物品的数量，同一样物品想拿多少拿多少，则问题称为无界或完全背包问题。

如果一件物品没有件数限制，那么我们可以取0、1、2、...至多可以取`C/w[i]`件，按照之前的分析，状态转移方程可以改写为

`f[i][j] = max( f[i - 1][j], f[i - 1][j - k * w[i]] + k * v[i] )`

其中`k`需满足`0 <= kw[i] <= j`，那么此时的时间复杂度就变成了O(nC*Σ(C/w[i]))，很明显可以对它进行优化。

先回想一下0-1背包问题中的两层循环，第一层为0至n-1，第二层从右至左C至w[i]，而这里从右至左更新的原因，是为了保证第 i 件物品的状态一定由第 i - 1 件物品的状态得来，也就是说，考虑第 i 件物品时，依据的是一个一定没有选中过 i - 1 件物品的结论，因此如果将第二层循环改为从左至右，由w[i]至C，就变成了选第 i 件物品时依然从已经拿过第 i 件物品的结论中递推，此时的状态转移方程可以写为：

```c++
f[i][j] = max( f[i - 1][j], f[i][j - w[i]] + v[i] )
                              ^
                            注意这里变成了i，我们不再需要k这个变量
```

于是我们可以写出解决完全背包问题的代码：

```c++
int complete_knapsack(vector<int> v, vector<int> w, int n, int C){
    vector<int> f(C + 1);
    for(int i = 0; i < n; i++){
        for(int j = w[i]; j <= C; j--)
            f[j] = max(f[j], f[j - w[i]] + v[i]);
    }
    return f[C];
}
```

理解了上面的两个状态转移方程，就可以利用0-1背包问题的解决思路，顺利解决完全背包问题。

### 多重（有界）背包问题

如果限定物品`i`最多只能拿`m[i]`个，则问题称为有界或多重背包问题。

类似的，此时的状态转移方程可以写为：

`f[i][j] = max{f[i - 1][j - k * w[i]] + k * v[i] | 0 ≤ k ≤ m[i]}`

此时的时间复杂度是O(C·Σ(m[i]))。有一种将问题简化的方法，将这个问题转化为物品数量为 Σ(m[i]) 的0-1背包问题，但此时的时间复杂度还是O(C * Σ(m[i]))。我们尝试用二进制的思想来对它进行优化，将有m[i]个的第 i 件物品分成 k + 1 组，每一组有一个系数，分别为 1, 2, 2^2, ···, 2^(k-1), m[i] - 2^k + 1（其中 k 为保证最后一项大于零的最大整数），现在原本有m[i]件的第 i 件物品，被分成了 log(m[i]) 件，每一件物品的价值和费用都乘以原来的系数倍，这样时间复杂度可以降为O(C·Σ(log(m[i])))。

这样分组可以保证原来0~m[i]中的每一个数都可以用新分组的系数组合而成，且不会超过m[i]，下面简单的证明一下：
首先看系数中除了最后一项的所有项，每一项都是2的次方，也就是说如果用二进制表示用它们组合可以得到0~2^k - 1中的所有数，这样我们还剩下2^k~m[i]中的数需要表示，证明完成了一半；现在把最后一个系数加进来，我们发现它加上之前系数可以表示的最大值2^k - 1后得到的正是m[i]，我们可以想象先将所有系数全部取走，得到的便是m[i]，然后根据选择丢弃不需要的系数，我们便可以得到m[i] - 2^k + 1~m[i]中的所有值（从1~2^k - 1依次丢)，那么现在就剩下0~m[i] - 2^k这些值需要表示，而现在只需要证明2^k - 1比m[i] - 2^k要大就行了，因为前面证明已经得到了0~2^k - 1之间的数，这个证明很简单，直接相减，然后利用m[i] - 2^k + 1 > 0的性质就可以得证。

证明了算法的正确性，就可以写代码了，这里我们把0-1背包和完全背包的第二层循环抽象成函数，然后用它们进一步定义`MultiplePack`，它同样是这种情况下第二层循环抽象出来的函数：

```c++
void ZeroOnePack(vector<int>& f, int wi, int vi, int C){
    for(int j = C; j >= wi; j--)
        f[j] = max(f[j], f[j - wi] + vi);
}

void CompletePack(vector<int>& f, int wi, int vi, int C){
    for(int j = wi; j <= C; j++)
        f[j] = max(f[j], f[j - wi] + vi);
}

void MultiplePack(vector<int>& f, int wi, int vi, int mi, int C){
    if(mi * wi >= C){ // 此时最多将物品拿空，相当于完全背包
        CompletePack(f, wi, vi, C);
        return;
    }
    // 其余的按0-1背包问题解决
    int k = 1;
    // 1, 2, ..., 2^(k-1)
    while(k < mi){
        ZeroPack(f, wi * k, vi * k, C);
        mi -= k;
        k *= 2;
    }
    // 此时mi已经变成了mi - (2^k - 1) = mi - 2^k + 1
    ZeroOnePack(f, wi * mi, vi * mi, C);
}
```

这里分析一下代码，以`mi = 13`为例，应分系数为1, 2, 4, 6四个，`while`循环中计算了1, 2, 4三个系数的情况，最后的6 = 13 - 2^3 + 1，而2^3 - 1正好等于1 + 2^1 + 2^2 = 2^3 - 1 = 7，循环中已经将它们减去，剩下的便是mi - 2^k + 1（高手，劳资怕是一辈子想不出这种写法）。

最后调用的时候将`MultiplePack`写进第一层循环中就行了：

```c++
vector<int> f(C + 1);
for(int i = 0; i < n; i++)
    MultiplePack(f, w[i], v[i], m[i], C);
```

#### 要求2：填满背包

背包问题还有另一种要求，即如何选取物品将容量为 C 的背包恰好填满（可能不存在）。

这时我们不必考虑物品的价值而只需考虑重量，规定`f[i][j]`表示用前 i 件物品填满容量为 j 的背包后，还剩下多少第 i 件物品可用，如果`f[i][j] = -1`则说明无解。

```c++
vector<vector<int> > f(n + 1, vector<int>(C + 1, -1)); // 初始化全部无解
// 注意这里物品数量初始化为 n + 1
f[0, 0] = 0; // 容量为0的背包不拿任何物品就可以填满，因此初始化为0
for(int i = 1; i <= n; i++){
    for(int j = 0; j <= C; j++)
        if(f[i - 1][j] >= 0)
            f[i][j] = m[i - 1];
        else
            f[i][j] = -1;
    for(int j = 0; j <= C - w[i - 1]; j++)
        if(f[i][j] > 0)
            f[i][j + w[i - 1]] = max(f[i][j + w[i - 1]], f[i][j] - 1);
}
return f[n][C];
```

这个算法的复杂度是O(nC)。然而这个算法目前没有看懂。

### 混合三种背包问题

如果物品中既有最多只能拿`m[i]`个物品，又有不限数量的物品，又有只有一件的物品，我们该如何选取呢？

这个问题看起来非常复杂，但是其实仔细分析一下，我们可以将前面的三种背包问题作为三种不同的情况来将复杂的问题简单化。这里就可以体现出我们前面将三种情况抽象成函数的意义了：

```c++
vector<int> f(C + 1);
for(int i = 0; i < n; i++){
    if(m[i] == 1)               // 为0-1背包
        ZeroPack(f, w[i], v[i], C);
    else if(m[i] == INT_MAX)    // 为完全背包
        CompletePack(f, w[i], v[i], C);
    else                        // 为多重背包
        MultiplePack(f, w[i], v[i], m[i], C);
}
```

这样就可以简洁明了的解决这个看似复杂的背包问题。

感谢《[背包问题九讲](https://github.com/tianyicui/pack)》中的详细讲解。