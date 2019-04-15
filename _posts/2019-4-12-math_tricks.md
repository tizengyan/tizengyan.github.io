---
layout: post
title:  "数学常用套路"
#date:   2019-02-26 17:11:54
categories: 算法
tags: mathematics
excerpt: 总结一下可能会用到的数学方法
author: Tizeng
---

* content
{:toc}

## 最大公约数

要找两个数字的最大公约数，常用的方法是**辗转相除法**。

举例来说，252和105的最大公约数是21（252 = 21 × 12; 105 = 21 × 5），因为 252 − 105 = 21 × (12 − 5) = 147 ，所以147和105的最大公约数也是21。在这个过程中，较大的数缩小了，所以继续进行同样的计算可以不断缩小这两个数直至其中一个变成零。

下面是代码实现：

```c++
int gcd(int a, int b) {
    int t = a % b;
    while(t != 0){
        a = b;
        b = t;
        t = a % b;
    }
    return b;
}
```

注意最后返回的是`b`而不是`a`，因为到最后`a`一定最后大于等于`b`，就算开始比`b`小，经过一轮取模就会交换。

再用递归实现一下，思路是一样的：

```c++
int gcd_recursion(int a, int b){
    if(a % b == 0)
        return b;
    return gcd_recursion(b, a % b);
}
```

## 最小公倍数

有了前面求得的最大公约数，最小公倍数就可以很容易的由它们的关系得出`lcm(a, b) = a * b / (gcd(a, b))`，就不放代码了。

## 求平方根

题目描述：输入一个正整数，输出它的平方根的整数部分。

这个题有两种思路，一是用二分查找，初始化`left = 1`，`right = INT_MAX`，然后按二分法的思路去找：

```c++
int mySqrt(int x) {
    if(x == 0)
        return 0;
    int left = 1;
    int right = INT_MAX;
    int mid = left + (right - left) / 2;
    while(mid != x/mid){
        if(mid > x/mid){
            right = mid - 1;
        }
        else if(mid < x/mid){
            if((mid + 1) > x/(mid + 1)) // 如果这个条件满足说明找到了整数部分
                return mid;             // 因为此时 mid + 1 的评分正好比 x 大
            left = mid + 1;
        }
        mid = left + (right - left) / 2;
    }
    return mid;
}
```

另一种思路就是用**牛顿法**（Newton's method），它是一种近似求解方程的方法，大致思路是要求解`f(x) = x^2 - a = 0`，先选一个较接近的点`(x0, f(x0))`，经过这个点且斜率为`f(x0)`导数的直线，与X轴的交点经推导为`x1 = x0/2 + a/(2*x0)`，这里`a`就是我们要求的答案，而每次迭代得到的结果都会一步步向`f(x)`真正的零点靠拢，因此只要输出第一次`Xn^2 < a`的值就行了。

下面是代码实现，十分简短，关键是要理解推导出的迭代公式：

```c++
int mySqrt(int x) {
    if(x == 0)
        return 0;
    long r = x;
    while(r > x / r)
        r = (r + x / r) / 2;
    return r;
}
```

注意这里`r`要声明为`long`型，因为如果`x`输入为`INT_MAX`，第一次进入循环便会溢出。