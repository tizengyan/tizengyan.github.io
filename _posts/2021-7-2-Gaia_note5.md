---
layout: post
title: "Gaia笔记5——CJ版本更新"
date: 2021-7-2
categories: UE4
tags: gaia
excerpt: 做的各种功能
author: Tizeng
published: false
---

* content
{:toc}

## 区分大厅和副本关卡（2021.7.2）

目前根据GameMode来区分

## Player初始化流程（2021.7.5）TODO

位置、碰撞、重力

玩家身上带的武器和技能在PlayerState中初始化，收到服务器消息回调后，先从GameMode拿到地图id，读地图表拿到默认携带武器id数组，根据该数组初始化人物身上的武器Actor，具体的逻辑是由Character上的HoldingItemComponent去做，Spawn武器蓝图之后用AttachToComponent挂到人物的Mesh上。技能的初始化也是类似。

技能和武器关联，每次切换武器后触发的回调中进行了上技能的操作，激活技能后会进入相应的GA文件中执行`ActivateAbility`，并给玩家上特定的标签，UI中就是靠注册标签移除和添加的事件来判断是否激活了技能。

## 主动技能UI优化（2021.7.6）

将之前蓝图的逻辑移植到了lua脚本，之前用“AsyncTask...”中的各种“ListenFor...”方法注册人物身上某个Tag的增加、移除和StackChange，蓝图中直接在后面拖线写逻辑就行，代码要从接口返回的实例中显示的注册该Task中定义的Delegate，和普通Delegate一样，用`Add`即可。主要遇到两个问题：

* 监听某个Tag的StackChange的时候，第一次返回的并不是当前的Count而是1，因为激活技能后是先走通知逻辑，再走的给属性，这样激活后第一次的数据就不对了。解决方法是初始化时覆盖第一次的同步，使用该技能对应的初始属性值（如最大值）。
* 每个人物属性有专门的方法拿，不管是通过Player还是ASC都是。为了能够在lua中方便的通过字符串拿属性，我写了一个执行字符串的方法当作宏使用。

## 搬运物（2021.7.16）TODO

## Bug合集

* 出现挂机之后大厅人物摄像头转移角度，之后操作会crash的问题，看日志是Controller中的PlayerCharacter为空，但此时并未掉线，通过在Controller的UnPossess处打断点发现Player由于掉出地图边界被销毁掉了。
