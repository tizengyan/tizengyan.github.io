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

## 1.合并两个排序的链表

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

## 2.反转一个链表

详见另一篇

## 3.链表的倒数第k个节点

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

## 4.链表中的循环（[LeetCode 142](https://leetcode.com/problems/linked-list-cycle-ii/)）

题目描述：给一个单向链表，判断其中是否含有循环，如果有，返回循环的起始节点，如果没有返回`NULL`。

这道题是找链表是否循环的升级版，前者只需要一个链表是否有循环，对于这种情况只需要一快一慢两个指针一直往后走，看看它们会不会中途相遇。

那么假设这里链表中有循环，记环前面的路程是`x`，环的周长为`y`，两个节点在距离环起点`r`处相遇，那么当它们遇见后，走过的距离会有如下关系

> 2(x + r) = x + r + n×y

`n`是一个非负整数，表示走得快的指针在和慢指针相遇前已经跑过的圈数，那么化简这个式子可以得到

> x = n×y - r

`x`的值其实就是我们要求的，有了它就可以得到环的起始位置，而有了上面的式子，解法就很明显了，我们等两个指针相遇后，把其中一个指针放回链表头结点，然后再让两个指针以相同的速度（每次前进一个节点）移动，它们最终就会在环的起始点相遇。

```c++
ListNode *detectCycle(ListNode *head) {
    ListNode* fast = head;
    ListNode* slow = head;
    if(head == NULL)
        return NULL;
    if(head->next == NULL || head->next->next == NULL)
        return NULL;
    while(fast->next != NULL && fast->next->next != NULL){
        fast = fast->next->next;
        slow = slow->next;
        if(fast == slow)
            break;
    }
    if(fast != slow)
        return NULL;

    fast = head;
    while(fast != slow){
        fast = fast->next;
        slow = slow->next;
    }
    return fast;
}
```

这种题目还有一类变形，给定两个链表的头结点，判断它们中的节点有没有相交，如果有则返回交点。

面对这个题目马上可以想到的一种解法就是将其中一个链表的首尾相接，这样问题就回到了找一个链表中的循环，然后用刚才的思路去解，完成后再把链表还原就行了。

如果只需要判断是否有交点，我认为只需要比较两个链表的最后一个节点是否为同一个，这样做的时间复杂度是O(n+m)。

### 与数组的联系

LeetCode上还有一题（[LeetCode 287](https://leetcode.com/problems/find-the-duplicate-number/)），题目描述是有一个大小为 n+1 的数组，其中储存 1~n 的整数，其中至少有一个数字是重复的，现在假设也只有一个重复的数字，但它或许重复了多次，要求找出这个重复的数字。

而这个解法就很骚了，题目要求既不能对原先数组做任何改动（read only），也就是说不能排序，又只能用O(1)的空间，也就是说不能用哈希表储存出现过的数字，看到这好像把所有思路都断了，但其实我们还有另一条线索，就是数组中储存的数字全都是 1~n 的整数，这个条件很容易被忽略。如此一来数组中储存的元素就不再单纯是数字，而可以看成下标，也就是说其他元素的地址，因此这其实是一个链表，而且是有一个循环的链表（因为只有一个重复的数字），那么我们的任务就变成了寻找链表的起始节点，和之前的代码类似，但这里要注意`slow`和`fast`不能初始化成同一个节点，否则无法进入第一个循环，也因此后面放回`fast`在起点时要往前一步，放在0而不是`nums[0]`，否则他们俩永远也遇不到。

下面是代码：

```c++
int findDuplicate(vector<int>& nums) {
    if(nums.size() == 1)
        return nums[0];
    int slow = nums[0];
    int fast = nums[nums[0]];
    while(slow != fast){
        slow = nums[slow];
        fast = nums[nums[fast]];
    }
    fast = 0;
    while(slow != fast){
        slow = nums[slow];
        fast = nums[fast];
    }
    return fast;
}
```