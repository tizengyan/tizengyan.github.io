---
layout: post
title: "UE源码学习——摄像机"
date: 2025-03-23
categories: UE源码学习
excerpt: 总结一下UE中摄像机相关的类和功能实现方式
author: Tizeng
---

* content
{:toc}


相机的流转有很多实现方式，可以定义一个相机模式的基类，其中包含一系列相机更新的数据和方法，每帧更新，然后根据不同的模式需要去派生不同的子类，在切换相机模式的时候使用不同的类型，根据类型之间的差别进行过度和更新。
还有一种方式是为每种相机的模式定义枚举，然后用一个**栈**来维护相机的状态，进入新状态时就入栈，状态结束就出栈。根据栈上枚举的类型，找到配置的数据，用来在更新和混合时使用。

## 基本类型

PlayerController中有一个CameraManager，在世界tick所有controller的时候，会通知该manager调用`UpdateCamera`。

- `APlayerCameraManager`：维护一个ViewTarget，方法`ProcessViewRotation`会在controller调用`UpdateRotation`时为其提供新的旋转信息
- `FTViewTarget`：包含POV，计算POV所需的Target（可能是Pawn也可能是Controller），以及储存玩家信息的PlayerState
    - `FMinmalViewInfo`：表示POV的类，包括位置、Rotation、FOV等信息
- `UCameraComponent`：包含相机相关的各种配置，可以按需挂在actor上，CameraManager在更新时，会调用ViewTarget的`CalcCamera`方法获取POV，如果该actor上存在这个组件（且配置为使用），则会使用这个组件的POV

* `UGameViewportClient`：持有一个`FViewport`，EngineTick时调用该Viewport的`Draw`，也就是由里到外，调用自己的`Draw`，其中会遍历当前World所有`ULocalPlayer`，通过LocalPlayer得到View（调用`CalcSceneView`方法，创建并返回`FSceneView`），在得到View的过程中，便是使用ViewTarget的数据，LocalPlayer通过持有的PlayerController持有的PlayerCameraManager，得到它身上目前缓存的ViewTarget信息（`GetCameraCachePOV`）
* `FViewport`：继承自`FRenderTarget`，根据平台不同，执行不同的渲染实现。持有一个`ViewportClient`，会在Draw函数中调用该ViewportClient的Draw
* `FSceneView`：从场景映射到屏幕空间的数据（A projection from scene space into a 2D screen region.）
* `FSceneViewFamily`（Context）：储存已有的View，通过RAII处理`FSceneView`的销毁

**输入处理**：在Controller的`SetupInputComponent`中绑定了左右和上下转动的输入，通过Controller中的delegate通知到Player身上的CameraControlComponent，使用Controller中的`AddYawInput`和`AddPitchInput`方法转动摄像机
（这两个都是Controller的方法，输入处理的入口在Controller，那为什么不直接在Controller中处理转动而要跳一层呢？），
它们会将输入乘上InputScale（通常为2.5，Pitch为-2.5）得到具体的度数，累积到`RotationInput`成员中，然后在客户端Tick中将变化值（delta）加到ControlRotation上。

## lyra中的相机

参考[知乎文章](https://zhuanlan.zhihu.com/p/602806113)。
