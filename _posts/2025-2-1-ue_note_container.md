---
layout: post
title: "UE源码学习——常用容器"
date: 2025-02-1
categories: UE源码学习
excerpt: 详细了解一下UE中常用容器的实现
author: Tizeng
---

`TMap`由`TSet`实现，`TSet`由`TSparseArray`实现，我们一步步来看。

## TArray

简单的数组，可能会在增加或删除元素时调整所占空间大小（内部调用`ResizeGrow`、`ResizeShrink`），增加元素的`Emplace`方法会先调用`AddUninitialized`方法添加一个未构造的元素，然后直接在目标位置用placement new构造一个对应类型的对象。`Add`本质就是调用了`Emplace`。

与普通数组一样，删除元素会把删除位置后面的所有元素往前移动，效率较低，如果不在意数组的顺序，可以使用`RemoveAtSwap`方法，它会先调用待删除元素的析构，然后将最后一个元素移动（先拷贝过去，然后析构旧的）到那个位置。

除此之外`TArray`还提供了将数组变为堆的方法，可以方便的构建最大（小）堆，用来处理数据。

## TSparseArray

使用上和普通数组没有不同，只是删除元素时不再会发生移动，因此效率很高，但被删除元素所占内存不会被释放。

内部是一个`TArray`和一个`TBitArray`，先来看看什么是`TBitArray`。

### TBitArray

顾名思义，是一个元素全部都是1bit的数组，专门用来保存状态，内部每个内存块实际上是4个字节，也就是32bit（`sizeof(uint32)`），对元素操作的index除以32，便得到实际内存块的序号，对32取模，便得到该内存块中具体要拿哪一位的数据，下面看一下方括号的实现：

```c++
#define NumBitsPerDWORD ((int32)32)
FORCEINLINE FBitReference operator[](int32 Index)
{
	check(Index>=0 && Index<NumBits);
	return FBitReference(
		GetData()[Index / NumBitsPerDWORD],
		1 << (Index & (NumBitsPerDWORD - 1))
		);
}
```

`FBitReference`可以直接转换为`bool`，内部保存了一个32bit内存块的引用和我们要取哪一位的`Mask`，通过简单的`(Data & Mask) != 0`就可以判断该位置是否有值。其构造的第二个参数有一个`Index & (NumBitsPerDWORD - 1)`的操作，这实际上是对`Index`进行了范围限制，让它不超过32，因为31的二进制就是`0001 1111`。

增加元素也是简单的位运算：

```c++
void SetBitNoCheck(int32 Index, bool Value)
{
    uint32& Word = GetData()[Index/NumBitsPerDWORD];
    uint32 BitOffset = (Index % NumBitsPerDWORD);
    Word = (Word & ~(1 << BitOffset)) | (((uint32)Value) << BitOffset);
}

int32 Add(const bool Value)
{
    const int32 Index = AddUninitialized(1);
    SetBitNoCheck(Index, Value);
    return Index;
}
```

### 实现

大体思路是用一个`TArray`储存数据，一个`TBitArray`用来标记该位置的元素是否有效（是否被删除），一个链表作为freelist，管理当前空位，删除元素时将那个位置的内存插入链表，添加时先看看freelist中有没有，如果有则从链表中取。

其内部的`TArray`类型是一个`union`，当有数据的时候就是稀疏数组元素的类型，当被删除时，变为一个结构，作为链表节点，持有前驱和后继的index，说是链表，其实不需要额外内存，只需要将空位的内存用index串联起来即可，找的时候直接通过index拿到地址，重新在上面构造元素。

```c++
template<typename InElementType,typename Allocator /*= FDefaultSparseArrayAllocator */>
class TSparseArray
{
    // ......

    /** Allocated elements are overlapped with free element info in the element list. */
    template<typename ElementType>
    union TSparseArrayElementOrFreeListLink
    {
        /** If the element is allocated, its value is stored here. */
        ElementType ElementData;

        struct
        {
            /** If the element isn't allocated, this is a link to the previous element in the array's free list. */
            int32 PrevFreeIndex;

            /** If the element isn't allocated, this is a link to the next element in the array's free list. */
            int32 NextFreeIndex;
        };
    };

    /**
        * The element type stored is only indirectly related to the element type requested, to avoid instantiating TArray redundantly for
        * compatible types.
        */
    typedef TSparseArrayElementOrFreeListLink<
        TAlignedBytes<sizeof(ElementType), alignof(ElementType)>
        > FElementOrFreeListLink;

    typedef TArray<FElementOrFreeListLink,typename Allocator::ElementAllocator> DataType;
    DataType Data;

    typedef TBitArray<typename Allocator::BitArrayAllocator> AllocationBitArrayType;
    AllocationBitArrayType AllocationFlags;

    /** The index of an unallocated element in the array that currently contains the head of the linked list of free elements. */
    int32 FirstFreeIndex = -1;

    /** The number of elements in the free list. */
    int32 NumFreeIndices = 0;

    // ......
};
```

每次元素被删除，调用完析构后，便将该位置作为新元素插入链表，作为表头，而每次插入元素时，先看看这个链表是不是为空（判断`NumFreeIndices`），如果非空则拿出第一个（`FirstFreeIndex`），下面放一段源码中的类型定义。

遍历的时候需要注意，由于中间可能存在空挡，因此如果使用index遍历，则每次都需要判断一下`IsValidIndex`（内部实现是查一下`AllocationFlags`），或者直接使用迭代器。

## TSet

储存一系列不重复的元素，内部使用了`TSparseArray`，增删查都是O(1)的复杂度。

key和index是两个不同的概念，index是数据数组（Elements）的下标

hash index

## TMap

