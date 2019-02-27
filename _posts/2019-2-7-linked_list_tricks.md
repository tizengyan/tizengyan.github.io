---
layout: post
title:  "链表的一些常见套路"
date:   2019-02-06 20:34:54
categories: 考点
tags: 链表 递归
excerpt: 合并、反转
author: Tizeng
---

* content
{:toc}

## 合并两个排序的链表

这题考察链表的常见操作：合并，有递归和循环两种写法。这个操作可以被用在归并排序链表中，在另一篇总结中也有提到。

这一题比想象中的要复杂，绝非简单的比较两个节点的值然后互相交换`next`指针，稍有不慎就会使链表信息缺失。

递归的写法比较简洁，但是也更巧妙。这里我们需要做的操作是，比较当前两个链表的节点，并将值较小的节点作为头结点，然后把之前合并好的链表的末端节点接上这个头结点，继续后面的比较，可以发现后面的操作和前面的一模一样。这里的base case是当其中一个链表到底或为`NULL`时，我们合并的结果当然就是另一个链表。
其实我还没有完全理解这种写法，怎么看怎么邪门，这就是递归的力量吧，熟练之后可以从旁人难以理解的角度将问题解决。

```c++
ListNode* Merge(ListNode* head1, ListNode* head2){
    if(head1 == NULL)
        return head2;
    else if(head2 == NULL)
        return head1;
    ListNode* temp;
    if(head1->val < head2->val){
        temp = head1;
        temp->next = Merge(head1->next, head2);
    }
    else{
        temp = head2;
        temp->next = Merge(head1, head2->next);
    }
    return temp;
}
```

循环写法中，我们维护的是一个指向新建节点`cur`的指针，它在每次比较后，会将`cur->next`（也就是前一个节点的`next`）指向值较小的那个节点，然后把`cur`和值较小的节点的指针向后移一位，由于`cur`始终指向的当前比较节点之前的一个节点，因此改变`cur->next`并不会对它们造成任何影响。
当其中一个链表的指针到达底端时，退出循环，但此时的算法还不能算结束，因为另一个链表后面可能还有信息，而这时如果`cur->next`刚好指向了到底的链表，我们就会丢失另一个链表剩余的节点，因此需要做一个简单的判断，将这种情况补全，即如果其中一个链表还没有到底，那么把`cur->next`指向这个链表，最后返回创建的新节点的下一个节点，就是我们的答案了。

```c++
ListNode* mergeTwoLists(ListNode* l1, ListNode* l2) {
    ListNode* head = new ListNode(0);
    ListNode* cur = head;
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
    return head->next;
}
```

## 反转一个链表

详见另一篇

## 链表的倒数第k个节点

题目描述：输入一个链表，输出该链表中倒数第k个结点。

这题考察链表的遍历技巧，我们可以用普通方法，先遍历一遍链表，得到长度n，然后得到`n-k+1`个位置的节点，就是倒数第k个节点。
但是这样效率不高，正确的做法应该是维护两个遍历指针，让其中一个先走`k-1`步，第二个指针再一起开始走，这样当第一个指针到达链表尾部时，第二个指针就刚好在倒数第k个节点的位置。

下面的实现代码：

```c++
ListNode* FindKthToTail(ListNode* head, unsigned int k) {
    if(head == NULL || k == 0)
            return NULL;
    ListNode* t1 = head;
    ListNode* t2 = head;
    while(k > 1){
        if(t1 == NULL)
            return NULL;
        t1 = t1->next;
        k--;
    }
    while(t1->next != NULL){
        t1 = t1->next;
        t2 = t2->next;
    }
    return t2;
}
```

这里要注意的是边界条件的判断，若输入指针为空或`k=0`都要返回`NULL`。