---
layout: post
title:  "UE4笔记2——一些源码"
date:   2020-07-21
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

## TSoftObjectPtr

UE管理UObject的一种方式，内部储存了对象的“Path”，可以在需要的时候去动态的加载资源，实现跨地图保存，具体还需要看资料整理。

需要在编辑器中通过吸取拿到场景中的Actor时，使用`TSoftObjectPtr<AActor>`而非`AActor*`，如果要在lua中使用，则需要在C++层使用Get之后再传给lua。

## MoveTemp

源码里很多地方都用到了这个方法，类似于C++中的`std::move`，它将传入对象的引用移除，然后将其类型转换成一个右值引用，不同的是如果传入了rvalue或是const对象，会编译不过。通常使用它是为了在传入或赋值后中断上一个引用了它的地方，防止由于残留的引用使外界能对其再做修改。在类的Move-Constructor中会使用到Move，因为Move-Constructor以右值引用作为参数，它在用右值初始化实例时被使用（只要被定义），可以免去重复复制的过程。

Move这个操作的本质是更换了（如一块内存的）所有权，它不对指针指向的资源做操作，而是去改变谁去指向那些资源。因此在使用Move之前我们一定要确定将被Move的对象在之后一定不会被使用，只会被销毁。(2022.1.23)注意是对象，不是内存，之前该对象对应的内存已经通过Move被给到了新的对象中，旧对象此时应该被销毁或赋值。

## Forward

UE同样提供了Forward的操作，和C++自己的`std::forward`等价

## UnLua插件
