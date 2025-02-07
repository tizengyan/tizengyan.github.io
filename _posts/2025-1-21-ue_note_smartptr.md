---
layout: post
title: "UE源码学习——智能指针"
date: 2025-01-21
categories: UE源码学习
excerpt: 详细了解一下UE智能指针的实现
author: Tizeng
---

## 智能指针

对c++的`shared`、`unique`、`weak`指针的重新实现，使用引用计数进行gc，但不能用在`UObject`上，`UObject`是一套独立的内存管理系统。

### MakeShareable和MakeShared的区别

源码注释中写道：

```
MakeShareable() - Used to initialize shared pointers from C++ pointers (enables implicit conversion)
MakeShared<T>(...) - Used to construct a T alongside its controller, saving an allocation.
...
- Prefer MakeShared<T>(...) to MakeShareable(new T(...))
```

这里说的额外一次内存分配指的是`MakeShareable`的调用中，分配要管理的对象和引用计数器的内存是分开的，需要两个`new`，而`MakeShared`是在同一块内存上做了这两件事，因此只需要分配一次内存。具体实现是通过类`TIntrusiveReferenceController`，这个名字就表明它将引用计数功能嵌入了管理对象中，它只有一个`TTypeCompatibleBytes`成员，这个类的作用是根据传入的类型自动占用相应大小的内存，然后在`TIntrusiveReferenceController`构造中使用placement new在上面直接构造管理的对象，引用计数功能则继承自基类`TReferenceControllerBase`。

`MakeShareable`则是接收一个管理对象的指针，然后返回一个TRawPtrProxy，它可以隐式转换成`TSharedPtr`或`TSharedRef`。它们在构造时会`new`一个引用计数器出来。

### TSharedFromThis类

继承`TSharedFromThis`类，使用`AsShared`和`SharedThis`方法得到智能指针，不同的是后者可以通过传递`this`指针，得到派生类的ptr（通过模板函数识别参数类型）。注意不能用在析构中，因为那时候对象已经开始被删除，`check`会不过。

它内部缓存了一个指向自身的`WeakPtr`，前面说过TWeakPtr是不能单独使用的，一定是从共享的ptr或者ref转换得到，因此。

### TSharedPtr和TSharedRef

为了方便，下面统一用ptr和ref代指这两个类型。

ref必须为非空，或由一个非空ptr转换而来。ptr可以由ref隐式转换，反过来不行，需要调用`ToSharedRef`，其中会检查有效性，如果为空则不会通过编译。`ToSharedRef`中调用的是ref以ptr为参数的拷贝构造和移动构造，但这两个构造函数都被声明为了`private`，防止用户直接调用。而为了让`ToSharedRef`能调用私有构造，ptr被声明为了ref的一个`friend`。

### TWeakPtr

`TWeakPtr`是不能直接使用的，必须依赖于`TSharedPtr`，每次使用前都需要调用`Pin`方法得到一个`TSharedPtr`，判空后使用。同样的，为了调用`TSharedPtr`的私有构造，`TWeakPtr`也是`TSahredPtr`的一个`friend`。

其中的关键步骤是`TWeakPtr`尝试用自身的弱引用计数器构造`TSharedPtr`的共享计数器：

```c++
/** Creates a shared referencer object from a weak referencer object.  This will only result
    in a valid object reference if the object already has at least one other shared referencer. */
FSharedReferencer( FWeakReferencer< Mode > const& InWeakReference ) : ReferenceController( InWeakReference.ReferenceController )
{
    // If the incoming reference had an object associated with it, then go ahead and increment the
    // shared reference count
    if( ReferenceController != nullptr )
    {
        // Attempt to elevate a weak reference to a shared one.  For this to work, the object this
        // weak counter is associated with must already have at least one shared reference.  We'll
        // never revive a pointer that has already expired!
        if( !ReferenceController->ConditionallyAddSharedReference() )
        {
            ReferenceController = nullptr;
        }
    }
}
```

方法`ConditionallyAddSharedReference`会保证只有在共享引用计数大于0时，才会返回`true`，并增加计数，成功构造出有效的`TSharedPtr`，这个下面会详细讲到。

### 引用计数的实现

大部分实现可以在`SharedPointerInternals.h`和`SharedPointer.h`中找到。

主要功能在基类`TReferenceControllerBase`中，它有一个`SharedReferenceCount`用来记录强引用数量，和一个`WeakReferenceCount`记录弱引用数量，初始值都是1，当强引用数量降为0时，调用虚函数`DestroyObject`释放管理的对象。一开始可能会疑惑，既然弱引用不影响管理对象的生命周期，为什么还要记录弱引用数量？其实这是为了控制计数器本身的生命周期，计数器在创建时也是在堆上分配内存（前面提到过new一次和两次的问题），这是为了能让计数器在不同的智能指针中传递，维护同一个对象的引用计数，有分配就要有释放，弱引用数量就是计数器本身的释放条件，当弱引用数量降为0时，计数器会调用`delete this`将自己释放。

智能指针并不直接使用计数器，而是做了一层封装，分别是`FSharedReferencer`和`FWeakReferencer`，其中持有一个计数器指针，初始化时为`nullptr`，计数器在不同的智能指针中传递时，会按需调整强引用数量或弱引用数量，智能指针在销毁时会减少引用数量，其中要注意的是，当管理对象被销毁时，弱引用计数也要减1，这是因为在初始化时，弱引用也被初始化为1，可以视为弱引用中有一个位置是用来记录有无强引用的，部分代码和注释如下：

```c++
// Number of weak references to this object.  If there are any shared references, that counts as one
// weak reference too.  When this count reaches zero, the reference controller will be deleted.
//
// This starts at 1 because it represents the shared reference that we are also initializing
// SharedReferenceCount with.
RefCountType WeakReferenceCount{1};
```

根据派生计数器的不同，会有不同的释放管理对象内存的方式。
`TIntrusiveReferenceController`上面提到过，它将被管理的对象分配在了自己一个成员的地址上，因此并不需要额外调用`delete`去释放内存，只用调用管理对象的析构函数就好了，因此它的`DestroyObject`实现如下：

```c++
// in TIntrusiveReferenceController
virtual void DestroyObject() override
{
    DestructItem((ObjectType*)&ObjectStorage);
}
// in TIntrusiveReferenceController end

/**
 * Destructs a single item in memory.
 *
 * @param    Elements    A pointer to the item to destruct.
 *
 * @note: This function is optimized for values of T, and so will not dynamically dispatch destructor calls if T's destructor is virtual.
 */
template <typename ElementType>
FORCEINLINE void DestructItem(ElementType* Element)
{
    if constexpr (!TIsTriviallyDestructible<ElementType>::Value)
    {
        // We need a typedef here because VC won't compile the destructor call below if ElementType itself has a member called ElementType
        typedef ElementType DestructItemsElementTypeTypedef;

        Element->DestructItemsElementTypeTypedef::~DestructItemsElementTypeTypedef();
    }
}
```

`TReferenceControllerWithDeleter`会持有一个TDeleterHolder，根据传入的deleter来生成，并最终调用，如果没有指定则会使用`DefaultDeleter`，它会直接调用`delete`：

```c++
// in TReferenceControllerWithDeleter
virtual void DestroyObject() override
{
    this->InvokeDeleter(Object);
}
// in TReferenceControllerWithDeleter end

/** Deletes an object via the standard delete operator */
template <typename Type>
struct DefaultDeleter
{
    FORCEINLINE void operator()(Type* Object) const
    {
        delete Object;
    }
};
```

函数`NewDefaultReferenceController`会直接返回一个使用DefaultDeleter的计数器。

#### 线程安全

多线程这块还不是很熟悉，平时用的也很少，后面有需要再来填坑。

### TUniquePtr

最后说一下`TUniquePtr`的实现，它保证的是对一个对象独立的所有权，即对象的生命周期和它相绑定，这意味着它不能被拷贝（拷贝构造和拷贝赋值元运算符都被标记为delete），只能move或通过普通指针构造。可以调用`MakeUnique`直接构造对象并得到一个`TUniquePtr`。

同样可以自定义deleter来控制删除的逻辑。
