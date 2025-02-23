---
layout: post
title: "UE源码学习——反射"
date: 2025-02-16
categories: UE源码学习
excerpt: 总结一下UE中反射的实现原理
author: Tizeng
---

反射的定义是能让程序在运行时知道类型信息，而不限于编译期，具体实现方法牵扯的内容很多，大钊的专栏有一系列文章专门讲虚幻是如何生成和维护各个类型的反射代码的，我这里并不打算将这些过程都梳理一遍，而是想从日常开发中会遇到的一些问题入手，看看虚幻是怎么解决这些问题的。

## UClass是怎么表现类型信息的？

### StaticClass

获取当前UObject类对应的`UClass`，每个UObject都有，查看`.generated.h`文件可以找到是在`DECLARE_CLASS`宏中定义的，实现是调用了全局方法`GetPrivateStaticClassBody`，先通过`FindObject`看看能不能用提供的outer和name找到对应的`UClass`，如果找不到则new一个新的出来并初始化。

UClass自己调用StaticClass会返回什么？大钊的文章提到会引用到自身，这点还需要看下代码求证。

在引擎启动开始初始化时，会为每个UObject子类调用`StaticClass`创建`UClass`

## UPROPERTY宏的作用

被这个宏标记的成员会在`.gen.cpp`中生成`NewProp_`开头的反射数据，并被加入到`PropPointers`中
如果是UObject成员变量会被当做被这个类引用？
在gc时不会被删除
如果在外部被删除则会自动置空

## 如何通过名字拿到类中的值

直接看一种可行的方式：

```c++
void GetPropertyValueByName(UObject* Object, const FString& PropertyName)
{
    // Ensure the object is valid
    if (!Object) return;

    // Get the UClass object of the UObject instance
    UClass* ObjectClass = Object->GetClass();

    // Find the property by name
    FProperty* Property = ObjectClass->FindPropertyByName(*PropertyName);

    if (Property)
    {
        // Found the property
        if (FIntProperty* IntProperty = CastField<FIntProperty>(Property))
        {
            int32 PropertyValue = IntProperty->GetPropertyValue(Property->ContainerPtrToValuePtr<void>(Object));
            UE_LOG(LogTemp, Log, TEXT("Property '%s' value: %d"), *PropertyName, PropertyValue);
        }
        else if (FFloatProperty* FloatProperty = CastField<FFloatProperty>(Property))
        {
            float PropertyValue = FloatProperty->GetPropertyValue(Property->ContainerPtrToValuePtr<void>(Object));
            UE_LOG(LogTemp, Log, TEXT("Property '%s' value: %f"), *PropertyName, PropertyValue);
        }
    }
}
```

上面的代码做了以下几件事：
1. 找到对应的`UClass`，调用`FindPropertyByName`通过名字拿到对应属性`FProperty`
2. 通过`ContainerPtrToValuePtr`从属性中拿到对应实例的
3. 确定属性的类型，然后调用`FProperty`中的`GetPropertyValue`，使用第二步得到的地址作为参数得到值

为了搞清楚上面发生的事情，首先要理解虚幻是如何处理反射数据的，4.25版本引入了`FProperty`代替`UProperty`类型，它继承自`FFiled`，不继承自UObject，让属性的保存更加轻量化，内存管理上也可以和UObject脱钩。属性通过链表保存在`UStruct`（`UClass`的基类）中，有一个`ChildProperties`成员作为表头，不过`FindPropertyByName`是通过另一个成员`PropertyLink`进行遍历查找，原理是一样的。

在生成反射数据代码时，有一个偏移值，通过`STRUCT_OFFSET`获取，它的实现如下：

```c++
#define offsetof(s,m) ((::size_t)&reinterpret_cast<char const volatile&>((((s*)0)->m)))
```

其中`s`是类，`m`是成员，这里用`0`地址转换成`s`类型的指针后，对`m`进行访问，然后将其转化为一个字节的`char`并取地址，得到的结果便是这个成员相对于所在类的偏移量。
第二步通过“容器”，也就是真正包含这个属性的对象的地址，和`FProperty`中储存的地址偏移成员`Offset_Internal`，便可以得到对应成员的地址，如果是数组，就再根据给定的下标和`ElementSize`进一步偏移。最后调用属性子类中的`GetPropertyValue`方法，将地址转换成所需要的指针，之所以不直接使用，一是为了避免用户进行指针类型的转换，二是可以在不同的属性子类中做一些特殊处理。

## TFieldIterator

这个问题实际上是一个类中如何去遍历所有的成员变量？

## 如何通过名字调用函数

`IMPLEMENT_CLASS`会使用名为`StaticRegisterNatives##TClass`的方法来绑定当前类中所有**Native**函数的函数名和函数地址，将这些信息注册到`UClass`的`NativeFunctionLookupTable`成员中去。

不管是什么函数，都会生成一个`UFunction`对象
