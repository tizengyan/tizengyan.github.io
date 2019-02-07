---
layout: post
title:  "剑指offer：反转链表"
date:   2019-02-03 9:11:54
categories: 题
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