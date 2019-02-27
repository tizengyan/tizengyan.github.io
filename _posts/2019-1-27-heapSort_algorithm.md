---
layout: post
title:  "堆排序总结"
date:   2019-01-27 16:12:54
categories: 基础算法
tags: heap
excerpt: 在数组中建立最大或最小堆，并用其性质排序
author: Tizeng
---

堆排序具有空间原址性，任何时候都只需要常数个额外的元素空间储存临时数据。

根据完全二叉树的定义和性质，对于由数组表示的完全二叉树，我们可以根据当前下标计算出其父亲节点的下标和左右孩子的下标：

```c++
// 得到父亲节点下标(0 base)
int parent(int i){
    return (i - 1) / 2;
}
// 得到左孩子下标
int leftChild(int i){
    return i * 2 + 1;
}
// 得到右孩子下标
int rightChild(int i){
    return i * 2 + 2;
}
```

接下来我们要用这些信息构建最大堆，首先需要一个下沉函数`siftdown`，然后再用其构建堆。注意下标从0开始的话要格外注意循环起始位置。

```c++
void siftDown(vector<int>& v, int i){
    int maxIndex = i;
    int l = leftChild(i);
    int r = rightChild(i);
    if(l <= v.size() - 1 && v[l] > v[maxIndex])
        maxIndex = l;
    if(r <= v.size() - 1 && v[r] > v[maxIndex])
        maxIndex = r;
    if(i != maxIndex){
        swap(v[maxIndex], v[i]);
        siftDown(v, maxIndex);
    }
}

void buildHeap(vector<int>& v){
    for(int i = (v.size - 2) / 2; i >= 0; i++)
        siftDown(i);
}
```

有了构建堆的函数后，我们就可以用堆来排序了，基本思路是，将输入的数组构成最大堆后，将根节点（即数组首元素）与末尾的节点交换，
然后忽略最后一个元素，将根节点`siftDown`，如此往复，这样最大的元素就会依次沉到数组底部，也就排好序了。

```c++
void heapSort(vector<int>& v){
    int size = v.size() - 1;
    buildHeap(v);
    while(size != 0){
        swap(v[0], v[size]);
        size--;
        sfitDown(v[0]);
    }
}
```

此时我们已经完成了堆排序的所有步骤，但其实还有一些堆的操作这里没有用到，比如上浮函数`siftUp`、插入元素`insert`、删除元素`remove`、提取堆顶元素`extractMax`。

```c++
void siftUp(vector<int>& v, int i){
    while(i != 0 && v[i] > v[parent(i)]){
        swap(v[parent(i)], v[i]);
        i = parent(i);
    }
}

void insert(vector<int>& v, int val){
    v.push_back(val);
    siftUp(v, v.size() - 1);
}

void remove(vector<int>& v){
    v[i] = INT_MAX;
    siftUp(i);
    extractMax(0);
}

int extractMax(vector<int>& v){
    int res = v[0];
    v[0] = v[v.size() - 1];
    v.pop_back();
    siftDown(v, 0);
    return res;
}
```