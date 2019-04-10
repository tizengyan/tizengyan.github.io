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

**这段代码有问题！！**

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

2019-4-10更新：

实时证明，上面这个算法有问题，虽然通过了[牛客网的测试](https://www.nowcoder.com/practice/459bd355da1549fa8a49e350bf3df484?tpId=13&tqId=11183&rp=1&ru=%2Fta%2Fcoding-interviews&qru=%2Fta%2Fcoding-interviews%2Fquestion-ranking&tPage=2)，但是没有通过[LeetCode 53](https://leetcode.com/problems/maximum-subarray/submissions/)的测试。

举例输入数组[8,-19,5,-4,20]，由上面的程序逻辑，初始化`cur=0`（这里其实应该用`sum`，否则很容易误导人，给阅读代码带来误读的可能，这也是个教训），8比max大，但cur为0，因此累加，cur和max都更新为8，然后-19，它不比max大，因此继续累加，cur=8-19=-11，max不变，然后是5，它不比max大，继续累加cur=-11+5=-6，也不必max大，继续看-4，累加cur=-10，然后发现最后的20比max和cur都大，且cur小于0，然后更新max为20，GG。

这个写法本身就让人不舒服，命名又会产生歧义，给阅读带来障碍，难以debug，以后写代码一定要朝清晰、易懂的思路去写，不然会给后续的工作带来诸多不便，这次影响到了一次笔试，让我在笔试途中停下来思考很久还得不出结果，又确信以前应该是留下了正确的笔记，因为通过了牛客网的测试，可见：

1. 让自己不舒服的代码即使通过了测试也要将其修改为清晰的版本

2. 牛客网的测试案例还有待改进，不能盲目迷信。

正确的做法应该是这样：

```c++
int maxSubArray(vector<int>& nums) {
    int sum = 0;
    int maxN = INT_MIN;
    for(int j = 0; j < nums.size(); j++){
        sum = max(sum + nums[j], nums[j]);
        maxN = max(maxN, sum);
    }
    return maxN;
}
```

其实思路很简单，根本不需要比来比去，遍历时更新`sum`，比较叠加了nums[i]的情况和没有叠加的情况，选大的，然后更新`maxN`，没了。