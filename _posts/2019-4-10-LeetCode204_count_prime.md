---
layout: post
title:  "LeetCode 204——Count Prime"
#date:   2019-02-26 17:11:54
categories: 题
tags: math
excerpt: 计算质数的数量
author: Tizeng
---

* content
{:toc}

原题请参考[LeetCode 204](https://leetcode.com/problems/count-primes/)。

输入一个非负整数`n`，输出比它小的所有质数的数量。

质数的定义是除了1之外，只能被1和自身整除的数字，那么最简单的方法显然就是对于所有比`n`小的数，都去除以比自身小的数看看能不能整除，由此累积质数的数量，但是这样做的效率显然是很低的，时间复杂度会为O(n^2)。首先想到的优化方法就是对于当前的数字 i ，不从 i - 1 开始除，因为显然不可能整除，应该从开根号后的结果加一开始往下除，或者从2开始，一直除到`sqrt(i) + 1`，这样可以在一定程度上优化时间复杂度，但是仍然比较慢。

正确的做法应该是建立一个大小为`n`类型为`bool`的表，初始化都为`true`，表示每个数字是否为质数，我们先默认所有数都是质数，然后从2开始，判断其是否为质数，如果是，则与其相乘的所有整数的结果只要小于`n`，就肯定不是质数，在表中就应被设为`false`，也就是说，我们从2开始，依次将所有不是质数的数字排除，后面在遇到的时候就不用去判断它是不是质数了。

下面是实现代码：

```c++
    int countPrimes(int n) {
        if(n < 2)
            return 0;
        vector<bool> table(n, true);
        int count = 0;
        for(int i = 2; i < n; i++){
            if(table[i]){
                count++;
                for(int j = 2; i * j < n; j++){
                    table[i * j] = false;
                }
            }
        }
        return count;
    }
```

题目还可能要求输出一个范围内质数的数量，如 [m, n] ，那么我们只需要调用这个函数两次，计算出小于n质数的数量和小于m质数的数量再将它们相减即可。