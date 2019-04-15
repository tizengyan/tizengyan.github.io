---
layout: post
title:  "最长子串问题"
#date:   2019-02-26 17:11:54
categories: 题
tags: mathematics
excerpt: 字符串算法题
author: Tizeng
---

* content
{:toc}

## 1.每个字符至少出现k次的最长子串（[LeetCode 395](https://leetcode.com/problems/longest-substring-with-at-least-k-repeating-characters/)）

题目描述：输入字符串`s`和整数`k`，输出`s`中最长的子串长度，其中每个字符都至少出现`k`次。

```c++

```

## 2.最长公共子串（[LeetCode 718](https://leetcode.com/problems/maximum-length-of-repeated-subarray/)）

题目描述：输入两个数组A和B，输出同时出现在A、B中的最长子序列的长度。

这题的思路是用动态规划，数组`f[i][j]`表示

```c++
int findLength(vector<int>& v1, vector<int>& v2) {
    if(v1.size() == 0 || v2.size() == 0)
        return 0;
    int n1 = v1.size();
    int n2 = v2.size();
    int res = 0;
    vector<vector<int> > f(n1, vector<int>(n2));
    for(int i = 0; i < n1; i++)
        if(v1[i] == v2[0])
            f[i][0] = 1;
    for(int i = 0; i < n1; i++)
        if(v1[0] == v2[i])
            f[0][i] = 1;
    for(int i = 1; i < n1; i++){
        for(int j = 1; j < n2; j++){
            if(v1[i] == v2[j])
                f[i][j] = f[i - 1][j - 1] + 1;
            res = max(res, f[i][j]);
        }
    }
    return res;
```

初始化`f[i][j]`的第一行和第一列，如果某个数组中的第一个元素与另一个数组中的某个元素相等，则初始化该位置为1。

## 3.最长无重复子串（[LeetCode 3](https://leetcode.com/problems/longest-substring-without-repeating-characters/)）

题目描述：输入一字符串，输出最长的不含重复字符的子串长度。

这道题很容易陷入思考的怪圈，一定要冷静，要找出最长的不含重复字符的子串，首先需要一个表，这个表储存出现过的字符**最近一次**看到的位置，同时维护左右两个指针（其实只有一个，因为右指针就是遍历时的下标），计算当前子串的长度，然后和已知的最大长度比较，如果比它大就更新。每次查看下一个新字符时，如果它之前没有出现过，那么直接更新表中的数据为当前下标，如果是出现了的字符，就应该更新左指针到**上次出现位置**的右边一格，但是要注意，当且仅当更新的值比旧值大时才能做这个操作，因为左指针只能前进而不能后退。这样只需要遍历一次字符串。

```c++
int lengthOfLongestSubstring(string s) {
    int n = s.size();
    if(n == 0)
        return 0;
    vector<int> tb(256, -1);
    int mx = -1;
    int start = 0;
    for(int i = 0; i < n; i++){
        if(tb[s[i]] != -1){
            start = max(start, tb[s[i]] + 1);
        }
        tb[s[i]] = i;
        mx = max(mx, i - start + 1);
    }
    return mx;
}
```