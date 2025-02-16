---
layout: post
title: "UE源码学习——常用容器"
date: 2025-02-1
categories: UE源码学习
excerpt: 详细了解一下UE中常用容器的实现
author: Tizeng
---

`TMap`由`TSet`实现，`TSet`由`TSparseArray`实现，而它又由普通的`TArray`实现，我们一步步来看。主要参考[知乎文章](https://zhuanlan.zhihu.com/p/386420743)和源码，使用的引擎版本为5.1。

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
    // 先清除目标位的数据，再做或运算
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

储存一系列不重复的元素，内部使用了`TSparseArray`和哈希表，增删查都是O(1)的复杂度。

首先拆解一下内部的结构:

* 稀疏数组`Elements`：类型为`TSetElement`
    它封装了：
    * `ElementType Value`：注意不是指针，因为直接通过动态数组管理内存，数组的allocator会负责分配TSetElement的内存，这里没必要再用指针增加额外的判断
    * `FSetElementId HashNextId`：哈希桶中下一个元素的位置
    * `int32 HashIndex`：哈希桶在`Elements`中的位置
* 哈希表`Hash`：可以看作是allocator分配的一个`FSetElementId`的数组，每次使用`GetAllocation`方法获取地址，然后使用偏移和方括号拿到数据
    * `FSetElementId`内部就是一个index，用来指示在`Elements`中的位置
* 哈希表大小`HashSize`：由于每次扩容都是翻倍，因此这个值是2的幂次，在取余的时候可以用位运算优化，直接和`HashSize - 1`做与运算

key和index是两个不同的概念，index是内部数据数组（`Elements`）的下标，是哈希桶的id，而key是哈希表用来计算哈希值的依据，对`TSet`来说就是加进来的元素本身，可以使用`KeyFuncs::GetSetKey(Element.Value)`得到，默认情况下用的是`DefaultKeyFuncs`：

```c++
/**
 * The base KeyFuncs type with some useful definitions for all KeyFuncs; meant to be derived from instead of used directly.
 * bInAllowDuplicateKeys=true is slightly faster because it allows the TSet to skip validating that
 * there isn't already a duplicate entry in the TSet.
  */
template<typename ElementType,typename InKeyType,bool bInAllowDuplicateKeys = false>
struct BaseKeyFuncs
{
	typedef InKeyType KeyType;
	typedef typename TCallTraits<InKeyType>::ParamType KeyInitType;
	typedef typename TCallTraits<ElementType>::ParamType ElementInitType;

	enum { bAllowDuplicateKeys = bInAllowDuplicateKeys };
};

/**
 * A default implementation of the KeyFuncs used by TSet which uses the element as a key.
 */
template<typename ElementType,bool bInAllowDuplicateKeys /*= false*/>
struct DefaultKeyFuncs : BaseKeyFuncs<ElementType,ElementType,bInAllowDuplicateKeys>
{
    // ...
};
```

可以看到`BaseKeyFuncs`中通过模板参数定义了元素和key的类型，`TSet`所使用的`DefaultKeyFuncs`将其都定义为了`ElementType`，因此它返回的key类型就是元素类型。然后就可以用`KeyFuncs::GetKeyHash`得到计算出的哈希值（具体的计算方式看用的是什么`KeyFuncs`以及哪个重载函数）。得到哈希值后就可以从哈希表（`Hash`）中拿到index了：

```c++
FORCEINLINE FSetElementId& GetTypedHash(int32 HashIndex) const
{
	return ((FSetElementId*)Hash.GetAllocation())[HashIndex & (HashSize - 1)];
}
```

由于做了取余操作，可能会发生冲突，即不同元素映射到了同一个哈希桶上，因此找得到index之后还要在哈希桶中逐个比较，才能找到正确的index，`FindId`方法中使用了`KeyFuncs::Matches`，它的默认实现就是使用了`==`操作符来判断是否找到了正确的元素。

### Emplace

和`TArray`一样，`Add`实际上就是调用了`Emplace`，它做了以下几件事：

1. 在`Elements`上请求一个位置（`AddUninitialized`），然后在上面构造输入的元素（实际被`TSetElement`持有）
2. 计算元素的哈希值，调用`TryReplaceExisting`看看是否存在重复的key，如果存在则会将其覆盖，然后删除刚刚添加的元素（可以在`KeyFuncs`中配置`bAllowDuplicateKeys`来控制是否允许重复key）
3. 检查哈希表大小看看是否需要扩容，如果是则`Rehash`，重建新大小的`Hash`，对所有元素执行一遍`LinkElement`，如果不是则直接对新元素调用`LinkElement`

这里`LinkElement`做的事是重点，直接看代码：

```c++
/** Links an added element to the hash chain. */
FORCEINLINE void LinkElement(FSetElementId ElementId, const SetElementType& Element, uint32 KeyHash) const
{
	// Compute the hash bucket the element goes in.
	Element.HashIndex = KeyHash & (HashSize - 1);

	// Link the element into the hash bucket.
	Element.HashNextId = GetTypedHash(Element.HashIndex);
	GetTypedHash(Element.HashIndex) = ElementId;
}
```

只有三句，先将哈希值取余得到桶id，桶其实就是一个链表，通过`GetTypedHash`可以从`Hash`中获取到桶的位置，它实际上是`Elements`的下标，因此二三句的作用相当于对一个链表进行操作，将新的元素指向之前的表头，然后把表头替换成这个新元素的位置，那么下次查找的时候就会先找到这个刚加进来的元素了。完成后新元素便添加进了`TSet`中。

当`HashSize`发生变化时，都要检查一次是否需要rehash，`Reset`方法不会改变大小只会清空元素和哈希表，因此不会触发rehash，而`Empty`则有可能，因为它除了清空元素还会指定要留多少空挡。

### Remove

直接看代码：

```c++
template<typename ComparableKey>
FORCEINLINE int32 RemoveImpl(uint32 KeyHash, const ComparableKey& Key)
{
	int32 NumRemovedElements = 0;

	FSetElementId* NextElementId = &GetTypedHash(KeyHash);
	while (NextElementId->IsValidId())
	{
		auto& Element = Elements[*NextElementId];
		if (KeyFuncs::Matches(KeyFuncs::GetSetKey(Element.Value), Key))
		{
			// This element matches the key, remove it from the set.  Note that Remove sets *NextElementId to point to the next
			// element after the removed element in the hash bucket.
			Remove(*NextElementId);
			NumRemovedElements++;

			if (!KeyFuncs::bAllowDuplicateKeys)
			{
				// If the hash disallows duplicate keys, we're done removing after the first matched key.
				break;
			}
		}
		else
		{
			NextElementId = &Element.HashNextId;
		}
	}

	return NumRemovedElements;
}

void Remove(FSetElementId ElementId)
{
	if (Elements.Num())
	{
		const auto& ElementBeingRemoved = Elements[ElementId];

		// Remove the element from the hash.
		for(FSetElementId* NextElementId = &GetTypedHash(ElementBeingRemoved.HashIndex);
			NextElementId->IsValidId();
			NextElementId = &Elements[*NextElementId].HashNextId)
		{
			if(*NextElementId == ElementId)
			{
				*NextElementId = ElementBeingRemoved.HashNextId;
				break;
			}
		}
	}

	// Remove the element from the elements array.
	Elements.RemoveAt(ElementId);
}
```

`RemoveImpl`先通过哈希值找到桶，然后在哈希桶中逐个查找需要删除的元素，`Remove`执行真正的删除，要注意`Remove`中先调用了`GetTypedHash`拿到`Hash`中的桶id引用作为遍历的起始点，后续则是获取的`Elements`中元素的`HashNextId`引用，这是因为如果第一次就找到了要删除的元素，则需要对`Hash`进行更新，将下一个元素的id设置为桶的表头，如果不是则更新**前一个**（id是下一个，但是实际上是从前一个元素上获取）元素的`HashNextId`到下一个元素id上，也就是从链表中删除当前元素，结束后再执行真正的删除操作，即从`Elements`中移除该元素。

## TMap

`TMap`使用了一个`TSet`储存数据`Pairs`，类型是一个只使用key和value的`TTuple`作为pair，等价于只有key和value两个成员的结构体，`TMap`中的操作实际上都是对这个`Pairs`在做操作。前面提到过`TSet`通过`KeyFuncs`的实现方式来控制哈希值的计算和元素到key的转换，`TMap`利用了这个性质，实现并使用了`TDefaultMapKeyFuncs`：

```c++
/** Defines how the map's pairs are hashed. */
template<typename KeyType, typename ValueType, bool bInAllowDuplicateKeys>
struct TDefaultMapKeyFuncs : BaseKeyFuncs<TPair<KeyType, ValueType>, KeyType, bInAllowDuplicateKeys>
{
	typedef typename TTypeTraits<KeyType>::ConstPointerType KeyInitType;
	typedef const TPairInitializer<typename TTypeTraits<KeyType>::ConstInitType, typename TTypeTraits<ValueType>::ConstInitType>& ElementInitType;

	static FORCEINLINE KeyInitType GetSetKey(ElementInitType Element)
	{
		return Element.Key;
	}

    // ...
};
```

可以看到它的元素类型是`TPair<KeyType, ValueType>`，`GetSetKey`返回的是元素中的`Key`成员，这样就保证了修改数据时是使用`Key`来计算哈希值。

`TSet`是由`TSparseArray`实现的，它自然是支持排序的，因此`TSet`也可以排序，只要排序完了之后rehash一下即可，那么`TMap`的排序自然也就很简单了，只需区分一下`KeySort`和`ValueSort`，分别使用元素`TPair`中的key和value作为比较对象即可。

### TTuple（TODO）

## 总结

稀疏数组通过在删除时不释放内存的方式，提高效率，内部直接使用一个`TArray`来存放数据，然后一个`TBitArray`来标记每个元素是否有效，它内部实际上就是储存的32位整型的元素，然后通过换算下标和位运算来设置某个位上面的标记（01）。被删除元素留下的空挡在下个元素被加入的时候会重新使用，实现方法是将数据数组的类型定义为一个`union`，当有数据时就是元素类型，无数据时变为一个链表的结点，通过保存前后index的方式来链接各个空挡，然后再维护一个表头的下标和一个链表的长度，就构建了一个链表作为freelist，每当有新元素申请加入的时候，先看看freelist是否为空，如果非空则取出表头的位置，在上面构造新的元素，同时将链表的后一位记为表头。

`TSet`内部使用了稀疏数组，实际上存储的是一个新定义的结构，除了输入的元素外，还缓存了下一个桶的下标，形成一个链表，每个元素实际上是一个链表的表头。每当哈希表的大小，也就是`TSet`的大小发生变化，都需要进行rehash，重新构建一次桶中的数据。
添加元素时先在稀疏数组中申请一个位置并构造，然后更新哈希表的信息，首先需要用一个key计算哈希值，`TSet`直接使用了元素本身作为key，计算出哈希值，通过这个值从另一个维护桶id的数组中拿到桶id，也就是稀疏数组中哈希桶的初始位置（链表的表头），然后将新元素的id作为新的表头插入其中，并更新桶id数组。如果有需要则直接重新构建哈希表。
查找元素则通过哈希值找到数组中的桶，也就是第一个元素的位置，看看是不是我们要找的元素，由于不同的元素可能得到相同的哈希值，也就是冲突，因此需要逐个比对，不是则通过储存的下一个元素的id继续去找，直到找到正确的元素。
删除元素时除了直接将稀疏数组中的数据删除，还需要将哈希桶的信息进行更新，如果是桶中的第一个元素，则需要更新表头到下一个元素id上，如果是后面的元素，则需要在链表上将这个元素id移除。

`TMap`内部使用一个键值对作为元素的`TSet`，通过重写使用的`KeyFuncs`类，将key而非元素本身作为哈希值的计算依据，大部分的操作都是直接对这个`TSet`进行。
