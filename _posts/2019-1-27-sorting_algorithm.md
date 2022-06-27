---
layout: post
title:  "快排、归并排序总结"
date:   2019-01-27 11:01:54
categories: 基础算法
tags: sorting
excerpt: 总结一下快速排序和归并排序的思路，以及应用
author: Tizeng
---

* content
{:toc}

## 快速排序（quick sort）

核心是`partition`分割函数，将数组以一个数字为中心，将比其小或相等的元素放在左边，比其大的放在右边，操作完成后返回结束时这个数字的下标，然后递归的进行这个步骤。

### 代码思路

```c++
// 注意最后交换和返回的都是j + 1而非j。
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
    if(left >= right)
        return;
    int mid = partition(v, left, right);
    quickSort(v, left, mid - 1);
    quickSort(v, mid + 1, right);
}

// 在需要的地方直接调用
quickSort(input, 0, input.size() - 1);
```

维护一个下标指针`j = left - 1`，选定数组范围内中最后一个元素为比较标准`x`，从范围最左边`left`开始往后遍历，如果当前元素比`x`小或相等，则交换`j`后面的一个元素和当前元素，并将`j`往后移一位；反之则不作操作。
遍历完成后交换末尾的元素和`j + 1`指向的元素，并返回`j + 1`。

（2022.3.8）现在看来上面的话就是直接翻译代码，没有意义还不如读代码，Partition的思路是给定范围后，选取左端或右端的元素作为中间的比较值，然后调整除它之外其他元素的顺序，最后再把中间值换到中间去，并返回下标。

### 时间复杂度分析

#### **最坏情况划分**

当`partition`划分的子问题包含了0个元素和n-1个元素，此时的时间复杂度为O(n^2)，比如当数组已经完全有序时。
递归式为 T(n)=T(n-1)+O(n)。

#### **最好情况划分**

当`partition`划分的两个子问题规模都不超过n/2时，此时的时间复杂度为O(nlgn)。

递归式为 T(n)=2T(n/2)+O(n)。

#### **平衡划分**

假设算法每次都产生9:1的划分，递归式为T(n)=T(9n/10)+T(n)+cn，最后的时间复杂度为O(nlgn)。事实上任何一种常数比例的划分都会产生深度为O(lgn)的递归树，期中每一层的时间代价都是O(n）。因此只要划分为常数比例，算法的复杂度总是O(nlgn)。

通常情况下，`partition`所产生的划分中同时混有好的划分和坏的划分，且在递归树中这两种情况随机分布。而坏的划分所产生的代价可以被吸收到好的划分中，因此当它们交替出现时，程序的时间复杂度仍为O(nlgn)。

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

#### 1. 数组中出现次数超过一半的数字（Find Majority）

数组中有一个数字出现的次数超过数组长度的一半，请找出这个数字。例如输入一个长度为9的数组{1,2,3,2,2,2,5,4,2}。由于数字2在数组中出现了5次，超过数组长度的一半，因此输出2。如果不存在则输出0。

这个题目至少有三种解法，利用`partition`函数的算法就是其中一种，时间复杂度为O(n)。大体思路为，如果将数列排好序，那么下标为n/2的数字必定是那个出现次数超过一半的数（如果存在）。因此，每次运行完`partition`后比较返回的下标与n/2的大小，如果小于n/2，说明中位数在右边，再对右边剩下的部分运行`partition`，如果大于n/2，说明中位数在左边，应对左边剩下的部分运行`partition`，而如果等于n/2，说明我们已经找到中位数，如果majority存在，则必定为该中位数。

下面是实现代码：

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

#### 2. 最小的k个数

输入n个整数，找出其中最小的K个数。例如输入4,5,1,6,2,7,3,8这8个数字，则最小的4个数字是1,2,3,4。

这道题最简单的思路为先将数组由小到大排好序，再输出前k个数字，这样的算法复杂度为O(nlgn)，但我们同样可以利用`partition`函数来解决这个问题。

和前面一题类似，我们直接对数组调用`partition`，如果返回的下标小于k-1，说明还要继续查找右边部分的元素，如果返回的下标大于k-1，说明还要继续查找左边的元素，如果等于k-1，则终止循环。当循环结束后，数组的前k个元素就是其中最小的k个数了。

下面是实现代码：

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

我们也可以用c++中的`multiset`容器来解决这个问题。`multiset`可以容纳重复的元素并且会自动排好序，我们只需要在对输入数组遍历时，判断其是否已有k个元素，如果还没有k个元素，则直接添加当前元素，如果已经有k个元素，则与末尾最大的元素进行比较，如果比末尾元素大，说明`multiset`中储存的就是当前最小的k个数，可以直接跳过检查下一个元素，而如果此时的元素比末尾的元素小，说明我们应该更新`multiset`，讲末尾的元素替换为当前元素，再继续遍历，直到结束。

下面是实现代码：

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

## 归并排序（merge sort）

分治模式在每层递归都有三个步骤：

1. 分解原问题为若干子问题，这些子问题都是原问题规模较小的实例；

2. 用递归的方式解决这些子问题，若子问题的规模足够小，则直接得出答案；

3. 合并这些子问题的解成原问题的解。

归并排序是典型的分治法（divide & conquer），核心是`merge`函数，它的功能是将两个已经排好序的序列合并成一个更大的有序序列，需要提一点的是这个函数的空间复杂度是O(n)，用以在排序时储存比较的数组。

实现代码如下：

```c++
void merge(vector<int>& v, int left, int mid, int right){
    vector<int> v1, v2;
    for(int i = left; i <= mid; i++)
        v1.push_back(v[i]);
    for(int i = mid + 1; i <= right; i++)
        v2.push_back(v[i]);
    v1.push_back(INT_MAX);
    v2.push_back(INT_MAX);
    int i = 0, j = 0;
    for(int k = left; k <= right; k++){
        if(v1[i] <= v2[j]){
            v[k] = v1[i];
            i++;
        }
        else{
            v[k] = v2[j];
            j++;
        }
    }
}

void mergeSort(vector<int>& v, int left, int right){
    if(left >= right)
        return;
    int mid = left + (right - left) / 2;
    mergeSort(v, left, mid);
    mergeSort(v, mid + 1, right);
    merge(v, left, mid, right);
}
```

归并排序时一定要注意`Merge`函数中处理的mid位置和`MergeSort`中是一致的，不然会出现进入`Merge`后范围内的数并不是有序的。

### 1.用归并排序将一个链表排序（[LeetCode 148](https://leetcode.com/problems/sort-list/)）

思路其实和排序数组一样，只是相应的操作会转变成链表的方式。比如我们无法通过下标来找一个链表的中间节点，需要设置两个遍历速度不同的指针，一个指针`slow`每次走一步，另一个指针`fast`一次走两步，这样在`fast`走到链表底端时，`slow`会刚好处于链表正中间。（注意，这里题目要求使用O(1)的空间复杂度）

代码实现如下：

```c++
/**
 * Definition for singly-linked list.
 * struct ListNode {
 *     int val;
 *     ListNode *next;
 *     ListNode(int x) : val(x), next(NULL) {}
 * };
 */
ListNode* sortList(ListNode* head) {
    if(head == NULL || head->next == NULL)
        return head;
    ListNode* fast = head;
    ListNode* slow = head;
    ListNode* prev = slow;
    while(fast != NULL && fast->next != NULL){
        fast = fast->next->next;
        prev = slow;
        slow = slow->next;
    }
    prev->next = NULL; // split the list
    ListNode* l1 = sortList(head);
    ListNode* l2 = sortList(slow);
    return merge(l1, l2);
}

ListNode* merge(ListNode* l1, ListNode* l2){
    ListNode* l = new ListNode(0);
    ListNode* cur = l;
    while(l1 && l2){
        if(l1->val < l2->val){
            cur->next = l1;
            l1 = l1->next;
        }
        else{
            cur->next = l2;
            l2 = l2->next;
        }
        cur = cur->next;
    }
    if(l1)
        cur->next = l1;
    else if(l2)
        cur->next = l2;
    return l->next;
}
```

这里有两点要注意：

1. 排序函数`sortList`中通过`fast`、`slow`两个指针找到中间节点后，还需要一个指针记录`slow`之前的节点，用来在`fast`走到底后将链表从`slow`处切断，否则排序函数会一直原地打转无法生效。

2. 合并函数`merge`的逻辑要尤其注意，稍有不慎就会导致链表信息丢失，这个功能的实现比想象中的要复杂。首先新建一个节点，作为合并后头结点的代替，然后维护一个指针`cur`，初始化指向我们新建的节点，在比较两个链表节点的值后将`cur->next`设为值较小的那一个节点，这样新节点就和较小的节点连上了，然后将`cur`后移一位，较小节点的链表遍历指针也往后移一位。一直重复这个操作，如果出现链表`l1`中的的某个节点一直比`l2`中的很多节点大，那么`l2`会一直往后移，直到找到比`l1->val`大的值或到达链表底端。

我尝试在不新建节点的情况下来合并，但是似乎反而会使问题更复杂，因此暂时采用这个方法。

还有一个潜在问题是用递归调用函数时可能会在stack中积累占用空间，空间复杂度有可能超过O(1)，这个问题还有待确认。

### 2.数组中的逆序对（[剑指offer 面试题36](https://www.nowcoder.com/practice/96bd6684e04a44eb80e6a68efc0ec6c5?tpId=13&tqId=11188&rp=1&ru=/ta/coding-interviews&qru=/ta/coding-interviews/question-ranking)）

题目描述：

在数组中的两个数字，如果前面一个数字大于后面的数字，则这两个数字组成一个逆序对。输入一个数组，求出这个数组中的逆序对的总数P。


这道题最直观的思路就是对每一个元素进行扫描，扫描时与后面所有其他元素进行比较，如果找到逆序对，则`count++`，但是这样的时间复杂度是O(n^2)，显然不是最优解。

这时我们可以考虑用归并排序的性质来解决这个问题。
归并排序不停将输入的数组平均分割，直至每个子数组只有一个元素，然后依次合并并排序，层层递归，这里我们只需要在运行归并排序时计算每次合并时第一个子数组相对第二个子数组有多少个逆序对然后累加起来就行了，注意由于合并时默认两个数组都是有序数组，因此如果`v1`中的某个元素`v1[i]`大于`v2`中的某个元素`v2[j]`，则说明存在`j + 1`个逆序对，这点要特别注意，代码中累加的是`j`，因为我们在两个待合并数组中为了方便都加入了`INT_MIN`这个标志位，如果不这么做就应该累加`j + 1`。
还有一点，在合并两个子数组时我选择先将它们的第一个元素初始化为`INT_MIN`，这样做的好处是可以不用讨论当其中一个数组为空的情况，但是要注意如果`j`等于0时，`v1`中如果还有剩余元素需要比较，它们都会大于`v2[0]`，这时不能进行累加。

代码实现如下：

```c++
long long count = 0; // 注意这里输入数组长度可能非常大，为防止整形溢出采用long long类型

int InversePairs(vector<int> data) {
    mergeSort(data, 0, data.size() - 1);
    return count % 1000000007;
}

void mergeSort(vector<int>& v, int left, int right){
    if(left >= right)
        return;
    int mid = left + (right - left) / 2;
    mergeSort(v, left, mid);
    mergeSort(v, mid + 1, right);
    merge(v, left, mid, right);
}

void merge(vector<int>& v, int left, int mid, int right){
    vector<int> v1, v2;
    v1.push_back(INT_MIN);
    v2.push_back(INT_MIN);
    for(int i = left; i <= mid; i++){
        v1.push_back(v[i]);
    }
    for(int i = mid + 1; i <= right; i++){
        v2.push_back(v[i]);
    }
    int i = v1.size() - 1;
    int j = v2.size() - 1;
    for(int k = right; k >= left; k--){
        if(v1[i] > v2[j]){
            v[k] = v1[i];
            i--;
            if(v2[j] != INT_MIN)
                count += j;
        }
        else{
            v[k] = v2[j];
            j--;
        }
    }
}
```