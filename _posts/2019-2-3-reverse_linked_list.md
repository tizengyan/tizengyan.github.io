---
layout: post
title:  "剑指offer：反转链表"
date:   2019-02-03 9:11:54
categories: 算法题
tags: 链表 递归
excerpt: 题目描述：输入一个链表，反转链表后，输出新链表的表头。
author: Tizeng
---

* content
{:toc}

题目描述：

输入一个链表，反转链表后，输出新链表的表头。

## 思路

### 1.递归

我们可以利用递归的特性，将链表从最后一个节点开始，依次修改下一个节点的`next`指针为反方向，再将前一个节点的`next`设为`NULL`。这样程序会自动从末尾的节点开始，将整个链表反转，并将指向末尾节点的指针逐层返回。

```c++
ListNode* ReverseList(ListNode* head) {
    if(head == NULL || head->next == NULL)
        return head;
    ListNode* temp = ReverseList(head->next);
    head->next->next = head;
    head->next = NULL;
    return temp;
}
```

### 2.循环

我们还可以用循环来做这道题，从第一个节点开始，依次将`next`指针掉转方向。`pre`、`cur`、`nx`分别记录前一个节点、当前节点和后一个节点。循环结束的条件是`cur`指向了`NULL`，因此结束后`pre`指向的就是反转之后的头结点。

实现代码如下：

```c++
ListNode* ReverseList(ListNode* head) {
    if(head == NULL || head->next == NULL)
        return head;
    ListNode* pre = NULL;
    ListNode* cur = head;
    ListNode* nx = head->next;
    while(cur){
        cur->next = pre;
        pre = cur;
        cur = nx;
        if(nx)
            nx = nx->next;
    }
    return pre;
}
```

## 总结

这题主要考察的是对递归的理解和应用，以及对链表操作的熟练程度。

## 反转链表2（[LeetCode 92](https://leetcode-cn.com/problems/reverse-linked-list-ii/submissions/)）

反转链表的扩展，要求只反转链表中的一部分节点。

这题我们可以先找到需要进行反转的两个节点，用之前的思路反转后，再接回去，这样最坏的情况下需要遍历两次，而**头插法**可以做到只遍历一次。它的本质就是当经过要反转的头节点时，依次将后面的节点挪到最前，到最后一个需要挪动的节点时就完成了反转。这个算法的关键在于两点：
    1.根据给定的两个序号正确的找到开始和停止反转的节点
    2.挪动节点时既要保存开始节点的前一个节点以正确的链接，还要更新挪动后的反转完毕的表头
