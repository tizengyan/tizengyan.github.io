---
layout: post
title:  "堆排序总结"
date:   2019-01-27 16:12:54
categories: 基础算法
tags: heap
excerpt: 在数组中建立最大或最小堆，并用其性质排序
author: Tizeng
---

堆排序具有空间原址性，任何时候都只需要常数个额外的元素空间储存临时数据。堆的性质为，给定任意的两个节点P和C，若P为C的父节点，则P的值要大于等于（小于等于）C的值，那么此堆被称为最大堆，反之则为最小堆。

我们通常用由数组所表示的完全二叉树来表示一个堆，根据完全二叉树的定义和性质，我们可以根据当前下标计算出其父亲节点的下标和左右孩子的下标：

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

接下来我们要用这些信息构建最大堆，其中最重要的步骤是下沉元素，这里有两种写法，先介绍第一种也是比较常见的一种：

```c++
// 这个方法本质上是把start处的元素沉底
void maxHeapify(vector<int>& v, int start, int end) {
	int p = start;
	int c = getLeftChild(p);
    // 找出p最大的孩子节点（如果有），如果p比其小，就下沉，直到到达叶子节点或p已是最大
	while (c <= end) {
		if (getRightChild(p) <= end && v[getRightChild(p)] > v[c]) {
			c = getRightChild(p);
		}
		if (v[p] < v[c]) {
			swap(v[p], v[c]);
			p = c; // 将子节点设为新的父节点，继续下沉
			c = getLeftChild(p);
		}
		else {
			break;
		}
	}
}

void heapSort1(vector<int>& v) {
	if (v.size() == 0)
		return;
	// 从最底端最右的有孩子的节点开始，构建最大堆
	for (int i = v.size() / 2 - 1; i >= 0; i--) {
		maxHeapify(v, i, v.size() - 1);
	}
	// 这里可以从大往小遍历，但有size更易读一点
	int size = v.size();
	for (int i = 0; i < v.size(); i++) {
		swap(v[0], v[size - 1]); // 把最大值放到末尾
		size--; // 减小堆的范围
		maxHeapify(v, 0, size - 1); // 把换上去的元素沉底
		// 在除了根节点其他子树都是最大堆时，运行完这一行就再次构建了一个最大堆
	}
}
```

堆排序的基本思路是，将输入的数组构成最大堆后，将根节点（即数组首元素）与末尾的节点交换，然后忽略最后一个元素，将换上去的根节点下沉，再次生成对最大堆，如此往复，这样最大、倒数第二大、倒数第三大等元素就会依次沉到数组底部，也就排好序了。

第二种写法将下沉函数`siftdown`写成递归形式，然后再用其构建堆。注意下标从0开始的话要格外注意循环起始位置：

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
    for(int i = (v.size - 2) / 2; i >= 0; i--)
        siftDown(i);
}
```

和前面一样，有了构建堆的函数后，我们就可以用堆来排序了：

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

此时我们已经完成了堆排序的所有步骤，其实这两种写法的不同之处只是一个用了循环一个用了递归，核心思想都是将元素按顺序沉底构建最大堆，然后在排序时维护一个`size`变量，使已经排过序的元素保持不变。

还有一些堆的操作这里没有用到，比如上浮函数`siftUp`、插入元素`insert`、删除元素`remove`、提取堆顶元素`extractMax`，下面是一些补充函数：

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
