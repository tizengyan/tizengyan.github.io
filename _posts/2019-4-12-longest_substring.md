---
layout: post
title:  "算法题——最长子串问题"
#date:   2019-02-26 17:11:54
categories: 算法题
tags: algorithm
excerpt: 字符串问题中比较常见的一些子串问题
author: Tizeng
---

* content
{:toc}

## 1.每个字符至少出现k次的最长子串（[LeetCode 395](https://leetcode.com/problems/longest-substring-with-at-least-k-repeating-characters/)）

题目描述：输入字符串`s`和整数`k`，输出`s`中最长的子串长度，其中每个字符都至少出现`k`次。

```c++

```

## 2.最长公共子数组LCS（[LeetCode 718](https://leetcode.com/problems/maximum-length-of-repeated-subarray/)）

题目描述：输入两个数组A和B，输出同时出现在A、B中的最长子数组的长度。（子序列似乎可以不连续）

我们先来看暴力解法，从头到尾遍历两个数组，每次检查最长**公共前缀**并记下长度即可。那么很明显其中有非常多的重复比较，时间复杂度达到`O(n^3)`，可以用动态规划的思想来优化，用数组`dp[i][j]`表示nums1[i]和nums2[j]的最长公共前缀，注意是前缀，不是子序列，因此只要`nums1[i] != nums2[j]`都有`dp[i][j] == 0`，否则`dp[i][j] = dp[i + 1][j + 1] + 1`，这就是我们的状态转移方程，由于每个状态是由后面的状态决定，这就说明要从最后开始往前填表，过程和其他dp题大同小异，为了方便，我们在声明dp表时可以多声明一列，这样就不用对边界条件做特殊处理了，下面贴上空间优化后的写法，只保存上一列的数据，空间复杂度可以降到`O(min(M, N))`：

```c++
int findLength(vector<int>& nums1, vector<int>& nums2)
{
    int res = 0, size1 = nums1.size(), size2 = nums2.size();
    vector<int> dp(size1 + 1);
    vector<int> last;
    for (int j = size2 - 1; j >= 0; --j)
    {
        last = dp;
        for (int i = size1 - 1; i >= 0; --i)
        {
            if (nums1[i] == nums2[j])
            {
                dp[i] = last[i + 1] + 1;
                res = max(res, dp[i]);
            }
            else
                dp[i] = 0;
        }
    }
    return res;
}
```

这题还有一个滑动窗口解法，由较短的数组错位的比较每个交叉范围内的子串。

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

## 4.最长递增子序列（[LeetCode 300](https://leetcode-cn.com/problems/longest-increasing-subsequence/)）

题目描述：给定一个长度为N的数组，找出其中最长的单调自增子序列的个数（**不一定连续**，但是顺序不能乱），例如给定一个数组{ 1,3,5,4,7 }，则其存在两个最长子序列{1 3 4 7}和{1 3 5 7}，因此输出2。

这题很明显要用动态规划来做，用`dp[i]`表示以`nums[i]`**结尾的**最长子序列。接下来是确定状态转移方程，既然我们要知道以`nums[i]`为结尾子数组的最长子序列长度，那么它肯定是`nums[i]`之前所有子数组（`dp[j]`）中最大的那个值，且当`nums[i] > nums[j]`时，再加1。我们从左往右填表，便可得到该数组所有情况的最长子序列的值，最后找出最大的即可。

```c++
int lengthOfLIS(vector<int>& nums)
{
    vector<int> dp(nums.size(), 1);
    int res = 1;
    for (int i = 1; i < nums.size(); ++i)
    {
        for (int j = 0; j < i; ++j)
        {
            if (nums[i] > nums[j])
                dp[i] = max(dp[i], dp[j] + 1);
        }
        res = max(res, dp[i]);
    }
    return res;
}
```

（2022.4.5）这是网上很多地方提供的比较标准的解法，我想了很久一直想不通，既然`dp[i]`表示的是以第i个元素结尾的最长LIS，那么`dp[nums.size() - 1]`就应该是最终答案，因为我们要的就是nums这个数组的最长LIS长度，这是不符合我们填表和找最终答案的逻辑的。仔细看了讲解，发现人家说的其实是以第i个元素结尾的LIS的长度，这区别可太大了。这告诉我们在设计状态转移方程时，不一定要直接得到题目的最终答案，可以先获得给定数据一部分很多子问题的答案，然后再从中找到最终解答。

不过这样并不高效，如果用二分查找来优化dp，将第二层循环从遍历换成搜索，那么可以降低时间复杂度到`O(N*LogN)`。为了能进行搜索，数组就不再保存LIS的长度，而是储存末尾的元素值：

```c++
int lengthOfLIS(vector<int>& nums)
{
    vector<int> dp;
    for (int i = 0; i < nums.size(); ++i)
    {
        int left = 0, right = dp.size() - 1;
        while (left <= right)
        {
            int mid = (left + right) / 2;
            if (dp[mid] < nums[i])
                left = mid + 1;
            else if (dp[mid] > nums[i])
                right = mid - 1;
            else
                break;
        }
        if (left < dp.size())
            dp[left] = min(dp[left], nums[i]);
        else
            dp.push_back(nums[i]);
    }
    return dp.size();
}
```

扩展：打印这个最长的子序列，如果有多个则打印第一个。这个不清楚有没有原题，马上想到的思路是维护一个二维数组，储存以第i个元素结尾的LIS，再根据最后获得的i来索引出答案。

我们再来看一个升级版，找出数组LIS的个数（[LeetCode 673](https://leetcode.com/problems/number-of-longest-increasing-subsequence/)）。

## 5.最长回文字串（[LeetCode 5](https://leetcode-cn.com/problems/longest-palindromic-substring/)）

这题一般来说是两个思路，一个是中心扩散算法，每检查新的字符就向两边查找最长回文，要同时考虑奇数和偶数的情况，它的好处是只用遍历字符串一次，空间复杂度为常数级，时间复杂度为`O(n^2)`。

另一种是动态规划，时间复杂度和上面一样，优化后的空间复杂度是`O(n)`，难点在于找到合适的状态转移方程，从最优子结构的角度出发，如果一个字符串是回文，那么在其两端增加一样的字符后，新的字符串也是回文，而字串可以用一个范围来确定，因此可以写出转移方程：

    dp[i][j] = s[i] == s[j] && (j - i + 1 <= 3 || dp[i + 1][j - 1])

由于一个只有一个字符的情况下肯定是回文，因此`dp[i][i] = 1`，下面就是填表的问题了，由于每个坐标依赖的是(i + 1, j - 1)的值（以i为行从上到下，j为列从左到右），也就是左下角的值，因此要按列来填表，不能按行。优化思路是只用一列数据，因为我们不需要前面得到的值了，和其他dp思路一样，就不赘述了。
