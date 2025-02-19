---
layout: post
title: "UE源码学习——UObject"
date: 2025-01-21
categories: UE源码学习
excerpt: 学习一下UE中最基本的类
author: Tizeng
---

作为UE中最基本的类，UObject提供了诸如gc、反射、网络同步等各种基础功能，需要深入了解一下。主要参考源码、大钊和quabqi的专栏。

## 内存管理方式

UE的gc策略是mark and sweep，即分为标记清除两个阶段，每隔一段时间检查所有UObject的可达性，首先有一个root set，它包含的对象总是被认为是可达的，然后根据引用关系可以找到所有可达的UObject，那么其余没有被检测到的或是显式被标记要删除的对象，都会在gc周期到来时被清理掉。
只有被标记为uproperty的UObject指针才会被视为有引用，并在gc后被自动置空，如果不想用uproperty，则可以使用`FWeakObjectPtr`或`TStrongObjectPtr`。

UObject有两层基类，一个`UObjectBaseUtility`，里面主要是一些常用的接口，再往上是`UObjectBase`，基础功能实现大部分在这里，它在构造时会做两件事：

1. 调用`AddObject`，将自己加入全局容器`GUObjectArray`中，但要注意数组中并不是直接储存的对象指针，而是`FObjectItem`，它除了持有对象指针之外，还有一个`SerialNumber`，在需要时通过特定方法获取，实际上就是一个不停自增的整数，这样可以保证不重复，`FWeakObjectPtr`内部就是通过id和`SerialNumber`来缓存对象，因为只看id的话，无法分辨那个位置上的对象是否已经被释放或是被其他对象所代替，有了这个`SerialNumber`，就可以准确判断出当前引用的对象是否还有效了
2. 调用`HashObject`，用名字计算出哈希值，然后将自身、outer、class等信息存入一个哈希表单例中

除了下标和序列号，`FObjectItem`中还储存了对象的各种状态标志`Flags`，它的类型是一个继承自`int32`的枚举，其中定义的值都是使用位偏移得到的，因此最多可以储存32种状态，使用位运算就可以方便的去写入或移除，`AddToRoot`就是修改了`GUObjectArray`中对应对象的`Flags`，增加了一个`RootSet`标记，调用之后无论这个UObject是否被引用，都不会被gc。

析构时会将自身从`GUObjectArray`中移除，而在更早的阶段`BeginDestroy`时，便会通过重命名对象为`Name_None`的方式来将自身从哈希表中移除。

### FUObjectArray

全局数据`GUObjectArray`对应的类，内部是一个`FChunkedFixedUObjectArray`，通过划分多个chunk，每个chunk中储存一部分UObject来管理，代码中写死了每个chunk中的元素是65536（`NumElementsPerChunk = 64 * 1024`），在初始化时（`UObjectBaseInit`）根据配置，计算出需要多少chunk，然后预分配相应的空间，然后用二维数组（指针）进行索引，也可以先只分配一部分，然后在数量增加时进行扩充，分配新的chunk。

UObject内部会缓存一个`InternalIndex`，它就是该对象在`GUObjectArray`中的下标，`GetUniqueID`返回的也是这个值，换句话说只要有这个id就可以从全局数组中拿到这个对象，前提是它还有效。

### FUObjectHashTables

这是一个包含了若干个哈希表的类，作为一个**单例**使用。其中最主要的当然是储存UObject自身对象的哈希表，它是一个`TMap`，我们知道为了解决冲突，可以用一个`TSet`来作为哈希桶保存哈希值相同的元素，但并不是所有时候都有冲突发生，如果直接生成`TSet`可能会占用不需要的空间，因此虚幻专门写了一个类`FHashBucket`作为桶来优化，它会根据传入元素数量，来判断是否需要创建一个`TSet`，如果只是两个以内的元素则直接储存在一个长度为2的本地数组中，超过两个时才去创建`TSet`，然后把之前的元素放进去。

从这个类的名字可以看出它包含不只一个哈希表，除了直接用名字算出哈希值进行存储的`Hash`，还有下面这些哈希表成员，分别对应不同的使用场景，如无特别提及均使用了上面的`FHashBucket`作为哈希桶：

* `HashOuter`：一个`TMultiMap`，由outer和名字计算出的哈希值，存储UObject中全局数组的`InternalIndex`
* `ObjectOuterMap`：直接以outer作为key储存对象，目的是可以直接遍历某个outer下面的所有UObject
* `ClassToObjectListMap`：以对象的`UClass`作为key储存，想要遍历某个类型的所有对象就可以从这里拿，`TObjectIterator`就是拿到了`FUObjectHashTables`的单例后，从中找到这个object list然后进行遍历
* `ClassToChildListMap`：和上面的object list类似，但是只储存类型为`UClass`的对象，以`GetSuperClass`作为key，因此可以找到某个类下面的子类，如果遍历的时候需要包含所有子类的对象，则会先从这里将获取所有子类`UClass`，然后再一个个的去object list中找

提一句`TObjectIterator`有一个类型为UObject的特化版本，直接继承自`FUObjectArray`内部的`Iterator`，因为如果要遍历所有对象，直接从全局数组中开始就行了，不需要通过哈希表。

## UPROPERTY宏

被这个宏标记的UObject指针成员变量会被当做被这个类引用？
在gc时不会被删除
如果在外部被删除则会自动置空

## 常用的类和接口

引擎中提供了一些为UObject设计的类和接口，常用的有下面几种。

### StaticClass

获取当前UObject类对应的`UClass`，每个UObject都有，查看`.generated.h`文件可以找到是在`DECLARE_CLASS`宏中定义的，实现是调用了全局方法`GetPrivateStaticClassBody`，先通过`FindObject`看看能不能用提供的outer和name找到对应的`UClass`，如果找不到则new一个新的出来并初始化。

UClass自己调用StaticClass会返回什么？

### NewObject

引擎中用来动态创建UObject的方法，理论上所有用户创建的UObject都需要用这个接口创建，它大致做了下面几件事：

1. 调用`StaticAllocateObject`分配内存，根据名字不同有不同的处理
    - 没有指定名字时，调用`MakeUniqueObjectName`创建一个唯一的name
    - 指定了名字则尝试通过它和outer等信息去哈希表中查找有无存在的对象，如果找到则将其析构，没找到就用`GUObjectAllocator`分配一块内存出来
    - 在目标内存上构造`UObjectBase`，此时便会将自己加入全局数组和哈希表中
2. 使用上面的内存地址初始化`FObjectInitializer`参数，然后调用UObject的默认构造

继承自UObject的类经常会看到参数为`FObjectInitializer`的构造函数，它是保存在在这个`UObject`对应的`UClass`中的一个函数指针`ClassConstructor`中，查看`GENERATED_BODY`宏的展开，发现其在生成的头文件中，会使用以下两个宏定义：

```c++
#define DEFINE_DEFAULT_CONSTRUCTOR_CALL(TClass) \
	static void __DefaultConstructor(const FObjectInitializer& X) { new((EInternal*)X.GetObj())TClass; }

#define DEFINE_DEFAULT_OBJECT_INITIALIZER_CONSTRUCTOR_CALL(TClass) \
	static void __DefaultConstructor(const FObjectInitializer& X) { new((EInternal*)X.GetObj())TClass(X); }
```

注意上面的`TClass`不是`UClass`的类型，而是输入的UObject类型，因此最后会调用输入UObject的构造，根据用户使用的是`GENERATED_BODY`还是`GENERATED_UCLASS_BODY`会定义不同版本的默认构造（5.1源码中用这两个宏都会定义出有`FObjectInitializer`的版本）。而在`UClass`头文件中可以找到一个模板函数：

```c++
/**
 * Helper template to call the default constructor for a class
 */
template<class T>
void InternalConstructor( const FObjectInitializer& X )
{ 
	T::__DefaultConstructor(X);
}
```

`UClass`初始化的时候便是用这个函数保存到了函数指针`ClassConstructor`中，。

可以看到UObject只提供了这两种构造函数，在调用`NewObject`之后也只会触发这两种。

### TWeakObjectPtr

weak顾名思义是一个弱引用，它不会阻止被引用对象的gc，前面提到过其内部缓存了id和`SerialNumber`，获取对象通过id从`GUObjectArray`中拿到item，然后判断其序列号是否和自身缓存的相匹配，如果是则返回item持有的UObject指针。有效性直接通过item上的`Flags`信息进行判断。

### TStrongObjectPtr

既然有弱引用，那么就会有强引用，只要这个引用还在，被引用的对象就不会被gc，它的实现方式是内部有一个用`TUniquePtr`缓存的`TInternalReferenceCollector`，看名字可以知道它是一个引用收集器，内部缓存了一个UObject实例指针，初始化时会按需为收集器分配内存，如果已经存在则替换其中缓存的对象指针。

但收集器自身并不会阻止对象被gc，真正起作用的是其基类`FGCObject`，注释中写道这个类是用来给非UObject注册gc的，它在构造时会初始化一个全局静态`GGCObjectReferencer`，并调用AddToRoot，因此这个对象不会被gc。然后将自身（`FGCObject`）交给它的一个数组中保存，并在析构时从中移除。注意这里被全局`GGCObjectReferencer`管理的是收集器`TInternalReferenceCollector`（因为它继承自`FGCObject`）而不是被引用的UObject，由于收集器被标记为root，gc时会被调用内部的方法添加其他需要被索引的对象，此时便会把数组中所有的`FGCObject`也调用一次`AddReferencedObjects`，收集器便会将缓存的UObject指针给到传递进来真正的ReferenceCollector，以避免被gc。

当被释放时，会将自身从全局的Referencer中移除，下次gc到来时便不会阻止其gc了。

### TObjectPtr

用来代替指向`UObject`的指针，大小相当于64bit的普通指针，在编辑器模式下可以选择懒加载，本质是封装了一个`FObjectPtr`，这里面存了一个`Handle`，直接保存一个地址，取值的时候用`reinterpret_cast`将其转换为UObject的指针类型。
