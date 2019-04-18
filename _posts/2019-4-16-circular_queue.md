---
layout: post
title:  "循环队列"
#date:   2019-02-26 17:11:54
categories: 数据结构
tags: data_structrues
excerpt: 循环队列的实现
author: Tizeng
---

* content
{:toc}

循环队列是一种队列的实现，它充分利用了连续数组的储存结构，如果队列的头部元素被pop，那前面的空间就空出来了，但加入元素时还是往尾部在加，如果我们把数组看成一个环，超出尾端范围的元素再按顺序放入数组开头，就能防止假溢出的发生。

假设队列的头指针为`front`，尾指针为`rear`(注意尾指针指向的不是最低端的元素，而是最底元素的后面一格，因为需要判断队列是否为空），数组大小为`n`，队列长度为`len`，由此我们有两个常见的下标关系：

> rear = (front + len) % n
>
> front = (rear + n - len) % n

当`(rear + 1) % n == front`时，代表队列已满。