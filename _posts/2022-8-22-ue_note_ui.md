---
layout: post
title: "UE源码学习——UI相关"
date: 2022-08-20
categories: UE源码学习
tags: UI
excerpt: UI、输入内容汇总
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
* Screen：以游戏运行窗口为坐标系的像素坐标，又叫PixelPosition（与ScreenSpacePosition不同，后者是Absolute坐标）
* Viewport：以游戏窗口为坐标系的原始坐标，与Screen坐标之间有一个Scale的对应关系
* local：相对于另一个ui的坐标，通常是CanvasPanel，因此与local有关的接口都要求提供一个Geometry，这个便是它相对于谁的信息

## 输入相关

每个Actor中都有一个`InputComponent`，UserWidget中也有，同时可能存在多个InputComponent，它们在PlayerController由存在一个数组中管理，称为`InputStack`，每个Component都可以设置优先级，优先级高的会先被处理（按Priority从小到大排列，从后往前处理），如果中途优先级发生变更，需要主动将栈的顺序更新。

无论是Actor还是UserWidget中的InputComponent，只有当它进入到我们PlayerController的InputStack中时才会生效，

以鼠标的输入为例，我把它分为几个阶段。

### 第一阶段：引擎收到输入传给PlayerInput

下面面这些内容全都发生在`FEngineLoop::Tick`中。

1. 平台层从设备接收设备输入，传给SlateApp（`FSlateApplication`是一个单例）
2. SlateApp将点击信息封装进`FPointerEvent`转发给`SViewport`，它再转发给`FSceneViewport`
3. `FSceneViewport`会把鼠标的输入进行累加，记为`MouseDelta`
4. 同一个Tick内，SlateApp收到`FinishedInputThisFrame`，然后`SViewport`和`FSceneViewport`依次收到处理输入的通知
5. `FSceneViewport`内包含一个`ViewportClient`，它负责拿到LocalPlayer从而通知PlayerController调用`InputAxis`，最后调用Controller中`PlayerInput`的`InputAxis`，此时正式开始处理输入，进入第二阶段

### 第二阶段：PlayerInput处理输入

PlayerInput中有一个`KeyStateMap`，记录的是每个Key对应的输入数值信息（KeyState）。
下面这些内容由Controller的PlayerTick发起，这里以Axis输入举例。

1. Controller构建当前的InputStack（`BuildInputStack`）并传给PlayerInput
2. PlayerInput处理`KeyStateMap`，将RawValue转换成Value（`MassageAxisInput`，处理死区的逻辑也在这里）
3. 遍历InputStack，对每个InputComponent，再遍历其所有的输入绑定（如`AxisBindings`）
4. 通过每个绑定的AxisName，找到它对应的一系列Key（`KeyMappings`）
5. 通过Key在`KeyStateMap`中找到对应的输入值（Value），并将所有相同AxisName下的值累加（`DetermineAxisValue`），得到最终的`AxisValue`
6. 将`AxisValue`与绑定的Delegate对应，整个InputStack遍历完成后，再依次用存入的值Execute所有Delegate

输入的数据使用完后要被清除，由于RawValue在每次遍历计算完之后都会被设为0，因此只要没有新的输入进来，Value的值也会一直是0。

### 第三阶段：PlayerController处理监听回调

这阶段没啥好说的，就是使用InputComponent的接口注册InputSettings中的Action和Axis，处理业务逻辑。

根据设定的`MouseCaptureMode`，会决定是否将`ViewportClient`设为Focus，这点要注意，会影响其他界面的Focus状态。这个设置是在`FReply`的`AcquireFocusAndCapture`里面做的。后面点击信息在Route的时候会从里面取用。

### GameViewportClient

## UI输入事件的监听

一般分两种，第一种是在非UIOnly输入模式下，使用`ListenForActionInput`接口使用UserWidget中的`InputComponent`，监听特定Action绑定按键的响应。还有一种便是直接蓝图（lua）覆写`OnKeyDown`或者`OnPreviewKeyDown`方法，接收按键输入。

方法一中我们通过控制该UI的Priority，和是否Block等来控制接收输入的层级，而在方法二中，情况略有不同。两个方法都会受引擎Navigation系统的影响，在有Navigation的情况下，手柄十字键、键盘方向键的输入不会被接收到
`OnPreviewKeyDown`可以接收全部输入，但是UI的子组件focus时同样会接收到输入，此时如果子组件也要接收输入，有可能发生冲突，如果通过返回`Handled`结果阻止输入继续处理，那么Navigation会停止工作，这可能不是我们想要的

* `FSlateApplication`：最先接收到平台给过来的输入，手柄、键盘和其他设备的输入，最后都汇聚于此统一处理。由SlateUser得到（`GetFocusPath`）路径后，先从上往下遍历（tunnel），处理`OnPreviewKeyDown`事件，如果得到的Reply没有标记为Handled，则重新从下往上遍历（bubble），继续处理`OnKeyDown`事件。处理摇杆和普通按键的方法有所不同，前者是`ProcessAnalogInputEvent`，后者是`ProcessKeyDownEvent`，其中都对Navigation做了处理。

* `FSlateUser`：根据官方注释，它是每个输入源的代表，每当有新的输入源接入，便会创建一个SlateUser，其中储存了WidgetPath信息

* `FWidgetPath`：包含当前所有大小Widget的一个路径，持有一个`Widgets`数组（类型为`FArrangedChildren`，中间做了层封装），通过不同的遍历策略依次触发当中所有Widget的输入事件，如`FTunnelPolicy`会从0遍历到底，而`FBubblePolicy`则从最末端往上遍历。0为最上层的root widget。

* `FArrangedWidget`：持有Widget的Gemetry和一个`SWidget`的引用，而这个引用的实际类型（对UMG来说）其实是`SObjectWidget`。

* `SObjectWidget`：持有UserWidget指针，在SlateApp处理输入时，最终会调用到这个类的`OnKeyDown`方法（如果`OnPreviewKeyDown`没有被Handle），它调用UserWidget中的对应版本之后，如果返回的结果是Unhandled，那么还会调用一次`SWidget`的版本，对Navigation进行处理，并最终返回Handled。也就是说，如果在`OnKeyDown`中返回了Handled，那么该widget的Navigation逻辑就不会走了。同理如果在`OnPreviewKeyDown`中返回了Handled，Navigation也不会走，因为此时根本就不会去调用`OnKeyDown`。需要注意的是，有一个配置（`FNavigationConfig`）用来判断输入的按键是否可以Nav，默认的按键是方向键和手柄的十字键，如果输入的按键在配置中存在，则会返回对应的Nav方向，方向如果有效，那么就会把方向告诉Reply（`SetNavigation`）并返回Handled。

* `FReply`：由事件返回给系统，该如何处理接下来的任务
