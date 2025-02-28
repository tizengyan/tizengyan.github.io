---
layout: post
title: "UnLua实现学习"
date: 2025-02-28
categories: Lua
excerpt: 
author: Tizeng
---

拿了git上最新的unlua源码之后才发现，新版本改了很多实现方法，之前很多人写的教程都对不上了，要多花点时间自己看了。

## 绑定

新版本中从`FLuaContext`改到了`FUnLuaModule`中，同样是继承`FUObjectArray`中的创建通知，在UObject被加入全局数组时调用`FLuaEvn::TryBind`进行绑定，有以下几个步骤：

0. 如果这个UObject的UClass没有注册，则进行注册。使用UClass的名字调用`PushMetaTable`，它会新创建一个该类型名字的mt，定义其元方法为`Class_Index`，还有一些其他方法，最后加到全局表`UE`中，这就是为什么lua中可以使用`UE`拿到每个UObject类型
1. 使用lua的`require`加载脚本，拿到脚本中所有函数的名字
2. 遍历UClass中的所有`UFunction`，识别出可以被蓝图重写的函数
3. 找到蓝图函数中和lua重名的`UFunction`，将其替换为`ULuaFunction`

## 获取UPROPERTY

在`FLuaEvn`构造的时候会创建全局表`UE`，在其中加入`NewObject`、`LoadObject`等全局方法。绑定过程中则为`UE`增加以每个UObject名字为key缓存了一份`metatable`，它在调用的时候会通过名字找到对应的`FProperty`，然后通过反射信息中的地址偏移获取值。

现在的问题是这个UObject名字对应的mt是怎么和lua关联起来的

## 调用函数
