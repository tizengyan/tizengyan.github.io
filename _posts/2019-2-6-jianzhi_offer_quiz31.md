---
layout: post
title:  "剑指offer：连续子数组的最大和"
date:   2019-02-06 15:32:54
categories: 题
tags: 动态规划 递归
excerpt: 题目描述：输入一个整形数组，有正有负，数组中一个或连续的多个整数组成一个子数组，求所有子数组中和的最大的值。要求时间复杂度为O(n)。
author: Tizeng
---

* content
{:toc}

题目描述：

输入一个整形数组，有正有负，数组中一个或连续的多个整数组成一个子数组，求所有子数组中和的最大的值。
要求时间复杂度为O(n)。

## 思路

这题最简单的思路便是暴力解法，找出所有子数组并求和，然后选择其中最大的返回，但这样做的时间复杂度是O(n^2)。

### 动态规划

我们可以尝试用动态规划的方式去分析这个问题，我们从第一个数字开始累积，往后扫描所有的数，同时记录当前所见的最大值，即累积的值超过记录的最大值时，更新我们的记录，而如果出现比记录的最大值还要大的数，则抛弃前面记录的值，重新从当前数字开始累积。下面用公式表达：

![formula](https://github.com/tizengyan/images/raw/master/dp_formula.png){:height="70%" width="70%"}

```c++
int FindGreatestSumOfSubArray(vector<int> arr) {
    if(arr.size() == 0)
        return -1;
    int max = INT_MIN;
    int cur = 0;
    for(int i = 0; i < arr.size(); i++){
        if(arr[i] > max && cur < 0){
            cur = arr[i];
            max = arr[i];
            continue;
        }
        cur += arr[i];
        if(cur > max)
            max = cur;
    }
    return max;
}
```

这里需要注意判断重置`cur`和`max`的条件，必须同时满足当前累积的值`cur < 0`且`arr[i] > max`时才能重置，否则即使当前的数字比之前累积的和要大，如果累积的值为正数，我们仍然应该继续累积。