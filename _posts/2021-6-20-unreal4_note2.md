---
layout: post
title: "UE4笔记2——一些源码"
date: 2021-07-21
categories: 引擎
tags: ue4
excerpt: 做功能的时候读到一些引擎里面的接口实现，记录学习一下
author: Tizeng
---

* content
{:toc}


## InvSprt

## FindLookAtRotation

## ProjectWorldToScreen

## 内存管理（2022.7.7）

### 资源加载（2022.7.6）

- FSoftObjectPath：包含资源路径AssetPathName，支支持硬盘中的资源
- FSoftObjectPtr：继承自`TPersistentObjectPtr<FSoftObjectPath>`，包含资源路径，基类中缓存了该资源的弱引用（WeakPtr），使用Get方法拿到，如果还没有缓存，则会使用FSoftObjectPath的`ResolveObject`方法去找
- TSoftObjectPtr：对`FSoftObjectPtr`做了一层封装，提供模板类型参数和蓝图接口，对应的是蓝图中**青色**的SoftObjectReference类型，引用可能加载也可能没加载的对象（目录下的资源文件），可以用`AsyncLoadAsset`方法拿到**蓝色**的ObjectReference
- TSoftClassPtr：同样对`FSoftObjectPtr`做了一层封装，但是让其看起来像一个TSubclassOf，对应的是蓝图中**粉色**的SoftClassReference类型，引用可能加载也可能没加载的类和蓝图，可以用`AsyncLoadClassAsset`拿到**紫色**的ObjectClassReference

上面这些Soft引用和Weak引用的最大区别是前者可以引用未被加载的资源，但会有一份Weak作为缓存，而Weak必须引用已经存在了的实例。

`AddUObject`和`AddRaw`一个区别就是前者存UObject引用的方式是WeakPtr，而后者是普通指针。

- LoadObject：根据路径加载UObject资源
- LoadClass：一般用来加载蓝图资源并返回UClass，内部调用了`LoadObject<UClass>`

需要在编辑器中通过吸取拿到场景中的Actor时，使用`TSoftObjectPtr<AActor>`而非`AActor*`，如果要在lua中使用，则需要在C++层使用Get之后再传给lua。

按[文档](https://docs.unrealengine.com/5.0/en-US/asynchronous-asset-loading/)中的描述，UProperty硬指针与`TSoftObjectPtr`主要的区别在于硬指针索引到的东西会立刻被加载，有时这并不是我们所希望的。

### 异步加载

使用`FStreamableManager`中的接口`RequestAsyncLoad`，它可以传入一个delegate，在加载完成之后通知我们。

## MoveTemp

源码里很多地方都用到了这个方法，类似于C++中的`std::move`，它将传入对象的引用移除，然后将其类型转换成一个右值引用，不同的是如果传入了rvalue或是const对象，会编译不过。通常使用它是为了在传入或赋值后中断上一个引用了它的地方，防止由于残留的引用使外界能对其再做修改。在类的Move-Constructor中会使用到Move，因为Move-Constructor以右值引用作为参数，它在用右值初始化实例时被使用（只要被定义），可以免去重复复制的过程。

Move这个操作的本质是更换了（如一块内存的）所有权，它不对指针指向的资源做操作，而是去改变谁去指向那些资源。因此在使用Move之前我们一定要确定将被Move的对象在之后一定不会被使用，只会被销毁。(2022.1.23)注意是对象，不是内存，之前该对象对应的内存已经通过Move被给到了新的对象中，旧对象此时应该被销毁或赋值。

(2022.1.28)move所作的事情就是把输入的参数类型转化为rvalue，它不做任何内存分配，不“移动”任何东西，甚至不能保证返回的内容允许被移动，因为传入的类型可能为const。

## Forward

UE同样提供了Forward的操作，和C++自己的`std::forward`等价，它和move类似，都可以返回rvalue，不同的是它不总是返回rvalue，在传入左值时返回的是左值引用，它的目的是保持传入参数的类型不在过程中改变。要理解这个操作，需要先理解引用折叠、万能引用和完美转发。

### 引用折叠（reference-collapsing）

（参考《C++ Primer第五版》16.2节），在模板编程时，左值引用虽然不能和右值引用绑定，但如果一个左值传入了一个参数类型为右值引用的**模板类型**参数，那么编译器会将模板类型推定为一个左值引用（如`int&`）。这种情况下可以看成是进行了引用和引用的绑定，之后只剩下一层引用，所以称作引用的collapse：

    X& &, X& &&, X&& & -> X&
    X&& && -> X&&

### 万能引用（universal reference）

这是在《Effecttive Modern C++》中提出的概念，用来通俗的和右值引用做区分，它背后的原理其实是引用折叠。它的定义是如果一个模板函数中的参数类型被声明为`T&&`且T是被推导的，或者一个对象以`auto&&`声明，那么这就是一个万能引用，注意这里**被推导**是必要的，如果外部显示指定了类型，那它就是右值引用了。万能引用会被右值初始化为右值引用，被左值初始化为左值引用。之所以叫它万能引用，正是因为它既能是左值引用又能是右值引用，取决于传入的类型，同时还会保留`const`、`volatile`等属性。

* 注意任何阻止了类型推导的操作都会使其从万能引用变回普通的右值引用，如`void f(const T&& param)`中的`const`。
* 如果一个函数采用值返回（return by value），且可以返回一个右值引用或万能引用**参数**时，我们可以使用`move`或`forward`来尝试在返回时调用move构造，提高性能。这个优化只适用于右值引用或万能引用参数对象，不适用于返回函数本地创建的对象，因为编译器在返回本地创建的对象时会自己进行优化，省去拷贝的内存分配，这被称为RVO（return value optimization），具体请看书中Item 25。
* 不要重载使用了万能引用作为参数的函数，因为它会匹配（抢夺）所有没有被其他重载函数完全匹配的调用，造成不确定的问题。

想了一下为什么需要右值引用，大概是因为函数定义时需要一个既能接收右值，又能接收左值且达到引用的效果的参数类型，而如果按以前的标准，要接收右值，参数类型如果还带引用，就必须是const，而这又会阻止我们对其进行修改，所以需要一个新的类型。

### 完美转发（perfect forwarding）

有了上面的前提，就可以理解`std::forward`在调用时为何能

## 反射

虚幻的反射系统支持通过类成员的名字和类的实例拿到该成员的值（或内容）
UProperty
FindProperyByName
ContainerPtrToValuePtr
CopyCompleteValue

## UnLua插件
