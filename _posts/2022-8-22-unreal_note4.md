---
layout: post
title: "UE4笔记4——UI相关"
date: 2022-08-20
categories: 引擎
tags: ue4
excerpt: 
author: Tizeng
---

* content
{:toc}

## 跨场景UI

UserWidget默认会在地图加载之后`RemoveFromParent`，是在`AddToScreen`时注册了全局通知`LevelRemovedFromWorld`，我们重写相关的方法，便可避免其跟随场景删除。然而在AddToScreen时，如果当前拿不到ViewportClient，承载自身的Canvas就无法添加进去，创建的任何东西都会被销毁。

为了使用和之前AHUD -> UUIControlComponent -> RootWidget相似的结构，跨场景ui的Root直接继承以前的RootWidget类，重写OnLevelRemovedFromWorld方法，在切换地图时不将自身移除，由UIManager保存其实例管理。

UserWidget的`Initialize`和`NativeContruct`的调用时机是不同的，前者在创建完后立刻会调，而后者则是要等到`TakeWidget`调用时才会被重写的该方法调用（`AddToScreen`时会调用`TakeWidget`）。

(2022.11.3)
跨场景ui带来新的问题，由于部分ui需要在切换场景时异步加载，先在TransitionLevel加载完毕后再进入目标Level，那么此时创建的ui就是跨场景ui，它需要由CrossMapRoot管理，而之前设想只有系统提示这一类的消息需要跨场景存在，因此CrossMapRoot的ZOrder比RootWidget高，这导致跨场景的ui呼出的其他界面必须也放在CrossMapRoot上，否则无法遮盖上一层，如果这么做，RootWidget就相当于失去了存在的意义，且切换场景时未删除的ui实例也不会清除。
解决方法是将普通的RootWidget放进CrossMapRoot中，重写删除普通Root的方法，将其从CrossMapRoot中移除。

Controller控制输入的InputComponent会在场景切换生成新Controller进行Swap时，调用SetPlayer触发初始化。

## UI坐标问题

* Absolute：以显示器上的像素为坐标系的坐标，如果有多个显示器会拼接扩展范围
* Desktop/Window：就是Absolute坐标
* Screen：以游戏运行窗口为坐标系的像素坐标
* Viewport：以游戏窗口为坐标系的原始坐标，与Screen坐标之间有一个Scale的对应关系
* local：相对于另一个ui的坐标，通常是CanvasPanel，因此与local有关的接口都要求提供一个Geometry，这个便是它相对于谁的信息

## 插件按钮

使用某个版本Unlua时发现蓝图界面没有预期显示生成lua文件模板的按钮，给开发带来不便。

`TCommand`

`TCommandList`

`MapAction`
