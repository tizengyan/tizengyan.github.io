---
layout: post
title:  "快排、归并排序总结"
date:   2019-01-27 11:01:54
categories: algorithm
tags: sorting
excerpt: 总结一下快速排序和归并排序的思路，以及应用
author: Tizeng
---

* content
{:toc}

## 快速排序

核心是'partition'分割函数，将数组以一个数字为中心，将比其小或相等的元素放在左边，比其大的放在右边，操作完成后返回结束时这个数字的下标，然后递归的进行这个步骤。

### 代码思路

```c++
int partition(vector<int>& v, int left, int right){
    int x = v[right];
    int j = left - 1;
    for(int i = left; i < right; i++){
        if(v[i] <= x){
            j++;
            swap(v[i], v[j]);
        }
    }
    swap(v[j + 1], v[right]);
    return j + 1;
}

void quickSort(vector<int>& v, int left, int right){
    if(left <= right)
        return;
    int mid = left + (right - left) / 2;
    quickSort(v, left, mid - 1);
    quickSort(v, mid + 1, right);
}

quickSort(input, 0, input.size() - 1);
```

维护一个下标指针'j = left - 1'，选定数组范围内中最后一个元素为比较标准'x'，从范围最左边'left'开始往后遍历，如果当前元素比'x'小或相等，则交换'j'后面的一个元素和当前元素，并将'j'往后移一位；反之则不作操作。
遍历完成后交换末尾的元素和'j + 1'指向的元素，并返回'j + 1'。


### 时间复杂度分析

#### 最坏情况划分

当'partition'划分的子问题包含了0个元素和n-1个元素，此时的时间复杂度为O(n^2)，比如当数组已经完全有序时。

递归表达式为 T(n)=T(n-1)+O(n)。

#### 最好情况划分

当'partition'划分的两个子问题规模都不超过n/2时，此时的时间复杂度为O(nlgn)。

递归式为 T(n)=2T(n/2)+O(n)。

#### 平衡划分

假设算法每次都产生9:1的划分，递归式为T(n)=T(9n/10)+T(n)+cn，最后的时间复杂度为O(nlgn)。事实上任何一种常数比例的划分都会产生深度为O(lgn)的递归树，期中每一层的时间代价都是O(n）。因此只要划分为常数比例，算法的复杂度总是O(nlgn)。

通常情况下，'partition'所产生的划分中同时混有好的划分和坏的划分，且在递归树中这两种情况随机分布。而坏的划分所产生的代价可以被吸收到好的划分中，因此当它们交替出现时，程序的时间复杂度仍未O(nlgn)。

### 随机化的快速排序（Randomized_Partition）

在讨论平均情况性能时，我们假设输入数据的所有排列都是等概率出现的，而在现实中这个假设并不一定成立，因此我们可以人为的在算法中引入随机性，使得算法对于所有的输入都能获得较好的性能。

```c++
int randomized_partition(vector<int>& v, int left, int right){
    int i = rand() % (right - left + 1) + left;
    swap(v[i], v[right]);
    return partition(v, left, right);
}
```

### 对其他问题的应用

#### 数组中出现次数超过一半的数字（Find Majority）

数组中有一个数字出现的次数超过数组长度的一半，请找出这个数字。例如输入一个长度为9的数组{1,2,3,2,2,2,5,4,2}。由于数字2在数组中出现了5次，超过数组长度的一半，因此输出2。如果不存在则输出0。

这个题目至少有三种解法，利用'partition'函数的算法就是其中一种，时间复杂度为O(n)。大体思路为，如果将数列排好序，那么下标为n/2的数字必定是那个出现次数超过一半的数（如果存在）。因此，每次运行完'partition'后比较返回的下标与n/2的大小，如果小于n/2，说明中位数在右边，再对右边剩下的部分运行'partition'，如果大于n/2，说明中位数在左边，应对左边剩下的部分运行'partition'，而如果等于n/2，说明我们已经找到中位数，如果majority存在，则必定为该中位数。

```c++
int MoreThanHalfNum_Solution(vector<int> numbers) {
    if(numbers.size() == 0)
        return 0;
    int i = partition(numbers, 0, numbers.size() - 1);
    int mid = numbers.size() / 2;
    int start = 0;
    int end = numbers.size() - 1;
    while(i != mid){
        if(i > mid){
            end = i - 1;
            i = partition(numbers, start, end);
        }
        else{
            start = i + 1;
            i = partition(numbers, start, end);
        }
    }
    int count = 0;
    for(int& n : numbers){
        if(n == numbers[i])
            count++;
    }
    return count * 2 > numbers.size() ? numbers[i] : 0;
}
```

其他的思路还有多数投票算法（Boyer–Moore majority vote algorithm），和随机取数（仅适用于majority一定存在的情况）。

#### 最小的k个数

输入n个整数，找出其中最小的K个数。例如输入4,5,1,6,2,7,3,8这8个数字，则最小的4个数字是1,2,3,4。

这道题最简单的思路为先将数组由小到大排好序，再输出前k个数字，这样的算法复杂度为O(nlgn)，但我们同样可以利用'partition'函数来解决这个问题。

和前面一题类似，我们直接对数组调用'partition'，如果返回的下标小于k-1，说明还要继续查找右边部分的元素，如果返回的下标大于k-1，说明还要继续查找左边的元素，如果等于k-1，则终止循环。当循环结束后，数组的前k个元素就是其中最小的k个数了。

```c++
vector<int> GetLeastNumbers_Solution(vector<int> input, int k) {
    vector<int> ans;
    if(input.size() == 0 || k <= 0 || k > input.size())
        return ans;
    int start = 0, end = input.size() - 1;
    int index = partition(input, start, end);
    while(index != k - 1){
        if(index < k - 1){
            start = index + 1;
            index = partition(input, start, end);
        }
        else{
            end = index - 1;
            index = partition(input, start, end);
        }
    }
    for(int i = 0; i < k; i++)
        ans.push_back(input[i]);
    return ans;
}
```

我们也可以用c++中的'multiset'容器来解决这个问题。'multiset'可以容纳重复的元素并且会自动排好序，我们只需要在对输入数组遍历时，判断其是否已有k个元素，如果还没有k个元素，则直接添加当前元素，如果已经有k个元素，则与末尾最大的元素进行比较，如果比末尾元素大，说明'multiset'中储存的就是当前最小的k个数，可以直接跳过检查下一个元素，而如果此时的元素比末尾的元素小，说明我们应该更新'multiset'，讲末尾的元素替换为当前元素，再继续遍历，直到结束。

```c++
vector<int> GetLeastNumbers_Solution(vector<int> input, int k) {
    vector<int> ans;
    multiset<int> s;
    if(input.size() == 0 || k <= 0 || k > input.size())
        return ans;
    for(int& n : input){
        if(s.size() != k){
            s.insert(n);
        }
        else{
            if(*s.rbegin() > n){
                s.erase(*s.rbegin());
                s.insert(n);
            }
        }
    }
    for(int n : s)
        ans.push_back(n);
    return ans;
}
```

## 归并排序

