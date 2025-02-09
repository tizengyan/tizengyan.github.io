---
layout: post
title: "UE源码学习——UObject"
date: 2025-01-21
categories: UE源码学习
excerpt: 学习一下UE中最基本的类
author: Tizeng
---

作为UE中最基本的类，UObject提供了诸如gc、反射、网络同步等各种基础功能，需要深入了解一下。

主要参考源码、大钊和quabqi的专栏。

## 内存管理

UE的gc策略是mark and sweep，即分为标记清除两个阶段，每隔一段时间检查所有UObject的可达性，首先有一个root set，它包含的对象总是被认为是可达的，然后根据引用关系可以找到所有可达的UObject，那么其余没有被检测到的或是显式被标记要删除的对象，都会在gc周期到来时被清理掉。
只有被标记为uproperty的UObject指针才会被视为有引用，并在gc后被自动置空，如果不想用uproperty，则可以使用`FWeakObjectPtr`或`FStrongObjectPtr`。

UObject有两层基类，一个`UObjectBaseUtility`，里面主要是一些常用的接口，再往上是`UObjectBase`，基础功能实现大部分在这里，在构造时会做两件事：

1. 将自己加入全局容器`GUObjectArray`中，但要注意数组中并不是直接储存的对象指针，而是`FObjectItem`，它除了持有对象指针之外，还有一个`SerialNumber`，在需要时通过特定方法获取，实际上就是一个不停自增的整数，这样可以保证不重复，`FWeakObjectPtr`内部就是通过id和`SerialNumber`来缓存对象，因为只看id的话，无法分辨那个位置上的对象是否已经被释放或是被其他对象所代替，有了这个`SerialNumber`，就可以准确判断出当前引用的对象是否还有效了
2. 调用`HashObject`，用名字计算出哈希值，然后将自身、outer、class等信息存入一个哈希表单例中

除了下标和序列号，`FObjectItem`中还储存了对象的各种状态标志`Flags`，它的类型是一个继承自`int32`的枚举，其中定义的值都是使用位偏移得到的，因此最多可以储存32种状态，使用位运算就可以方便的去写入或移除，`AddToRoot`就是修改了`GUObjectArray`中对应对象的`Flags`，增加了一个`RootSet`标记，调用之后无论这个UObject是否被引用，都不会被gc。

析构时会将自身从`GUObjectArray`中移除，而在更早的阶段`BeginDestroy`时，便会通过重命名对象为`Name_None`的方式来将自身从哈希表中移除。

### GUObjectArray

它内部是一个`FChunkedFixedUObjectArray`，通过划分多个chunk，每个chunk中储存一部分UObject来管理，代码中写死了每个chunk中的元素是65536（`NumElementsPerChunk = 64 * 1024`），在初始化时（`UObjectBaseInit`）根据配置，计算出需要多少chunk，然后预分配相应的空间，然后用二维数组（指针）进行索引，也可以先只分配一部分，然后在数量增加时进行扩充，分配新的chunk。

UObject内部会缓存一个`InternalIndex`，它就是该对象在`GUObjectArray`中的下标，`GetUniqueID`返回的也是这个值，换句话说只要有这个id就可以从全局数组中拿到这个对象，

### FWeakObjectPtr

weak顾名思义是一个弱引用，它不会阻止被引用对象的gc，前面提到过其内部缓存了id和`SerialNumber`，获取对象通过id从`GUObjectArray`中拿到item，然后判断其序列号是否和自身缓存的相匹配，如果是则返回item持有的UObject指针。有效性直接通过item上的`Flags`信息进行判断。

### FStrongObjectPtr

既然有弱引用，那么就会有强引用，只要这个引用还在，被引用的对象就不会被gc，它的实现方式是内部有一个用`TUniquePtr`缓存的`TInternalReferenceCollector`，它的功能

继承自`FGCObject`，

## 反射

反射的定义是能让程序在运行时知道类型信息，而不限于编译期，可以从最简单的几个问题入手。

### 如何通过名字拿到值



### TObjectPtr

用来代替指向`UObject`的指针，大小相当于64bit的普通指针，在编辑器模式下可以选择懒加载，本质是封装了一个`FObjectPtr`，这里面存了一个`Handle`，直接保存一个地址，取值的时候用`reinterpret_cast`将其转换为UObject的指针类型。
