---
layout: post
title: "UnLua实现笔记"
date: 2025-02-28
categories: Lua
excerpt: 新版本改了很多实现方法，之前很多人写的教程都对不上了，要多花点时间自己看了
author: Tizeng
---

UnLua的最新实现已经和之前的有一些区别，但大部分的教程还是基于老版本的实现，为了不混淆，在讨论的时候默认基于老版本，如果是新版实现会特别标明。

## 绑定

`FLuaContext`中注册了UObject创建的通知，每当`UObjectBase`创建便执行`TryToBindLua`进行绑定。
在`RegisterClass`方法中为对应的UClass进行注册，大概有以下几步（只会进行一次）：

1. 为传入的UClass及其所有父类创建`FClassDesc`
2. 创建使用`Class_Index`和`Class_NewInex`的元表，并对绑定的lua脚本进行设置
3. 使用`OverrideUFunction`覆写函数

### 新版

新版本中从`FLuaContext`改到了`FUnLuaModule`中，同样是继承`FUObjectArray`中的创建通知，在UObject被加入全局数组时调用`FLuaEvn::TryBind`进行绑定，有以下几个步骤：

0. 如果这个UObject的UClass没有注册，则进行注册。使用UClass的名字调用`PushMetaTable`，它会新创建一个该类型名字的mt，定义其元方法为`Class_Index`，还有一些其他方法，最后加到全局表`UE`中，这就是为什么lua中可以使用`UE`拿到每个UObject类型
1. 使用lua的`require`加载脚本，拿到脚本中所有函数的名字
2. 遍历UClass中的所有`UFunction`，识别出可以被蓝图重写的函数
3. 找到蓝图函数中和lua重名的`UFunction`，将其替换为`ULuaFunction`

在`FLuaEvn`构造的时候会创建全局表`UE`，在其中加入`NewObject`、`LoadObject`等全局方法。绑定过程中则为`UE`增加以每个UObject名字为key缓存了一份`metatable`，它在调用的时候会通过名字找到对应的`FProperty`，然后通过反射信息中的地址偏移获取值。

现在的问题是这个UObject名字对应的mt是怎么和lua关联起来的

## 获取UPROPERTY

由于注册了元方法`Class_Index`，它会尝试从元表中根据key获取PropertyDesc，然后从中通过反射信息拿到属性值。

## 调用UFUNCTION

`GetField`会判断获取的变量是否为属性，如果不是则将`Class_CallUFunction`入栈，它会找到对应的FunctionDesc并调用`CallUE`。

### 覆写函数

参考[UnLua解析（二）使用Lua函数覆盖C++函数](https://zhuanlan.zhihu.com/p/102356134)。

旧版的覆写原理是定义一个新的字节码`EX_CallLua`，然后在`OverrideUFunction`中将对应UFunction上的`Script`成员中输入这个字节码和`EX_Return`，这样该函数Invoke时就会执行`execCallLua`。
它会根据当前frame中的信息，找到对应的FunctionDesc，将lua函数和参数push进栈并执行调用，lua函数调用完毕后再对栈上的返回值进行处理。

需要注意的是引用类型参数的处理，lua中如果覆写的函数带有引用类型的参数，会被当成OutParam进行处理，UHT生成的实现在返回前会将引用类型的参数在**参数结构体**中进行赋值，而lua覆写的函数调用完毕后FunctionDesc也会为此做处理，而它的顺序是把引用类型参数push进栈，再push函数自己的返回值。
因此lua实现中需要先按顺序返回引用类型的参数，再返回函数自己的返回值，不然UnLua就把函数的返回值按照引用参数进行处理，导致最后拿到的函数返回值有误。

新版中会创建一个`ULuaFunction`，它继承自UFunction，专门处理lua的覆写和调用。
