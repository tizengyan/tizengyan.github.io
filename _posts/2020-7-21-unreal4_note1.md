---
layout: post
title:  "UE4笔记1——上手"
date:   2020-07-21
categories: 引擎
tags: ue4
excerpt: ue4入门笔记
author: Tizeng
---

* content
{:toc}

<head>
    <script src="https://cdn.mathjax.org/mathjax/latest/MathJax.js?config=TeX-AMS-MML_HTMLorMML" type="text/javascript"></script>
    <script type="text/x-mathjax-config">
        MathJax.Hub.Config({
            tex2jax: {
            skipTags: ['script', 'noscript', 'style', 'textarea', 'pre'],
            inlineMath: [['$','$']]
            }
        });
    </script>
</head>

## 名词

* LOD：细节层次（level of details），类似于mipmap，如果一个物体只占屏幕较小的区域，那么就减少它的顶点和面数来节省开销
* UMG（unreal motion graphics UI designer）
* AF：各向异性过滤（anisotropic filtering）
* mipmap：多级渐远（进）纹理
* sRGB：标准（standard）红绿蓝色彩空间，一般使用Gamma系数为2.2的色彩空间
* gamma correction：伽马校正，通常是输入进行$\gamma$次方再乘以一个常数（通常为1），如$V_{out}=A \cdot V_{in}$，伽马大于1时阴影会变得更暗，小于1时暗处会变得更亮
* DCC app：数字内容创作（digital content creator）软件
* Actor：场景中的物件，类似unity中的GameObject
* lightmap：光照贴图，它将复杂的光照和阴影信息储存进纹理中，这样就能在运行时以很低的开销显示出很好的效果

## 注意事项

* 总是使用长宽为2的幂次方的纹理图片，否则不会生成mipmap，HDR格式除外
* 嵌入式alpha的贴图导入ue4中时不会被压缩，因此会比分离式alpha消耗更大的空间（大概两倍），而且如果降低嵌入式alpha的质量，其他颜色通道的质量也会被降低，分离式alpha则不会有这个问题，即在基本保留原有观感的情况下降低alpha通道的开销
* 纹理在作为遮罩（Mask）时应该不勾选sRGB以取消Gamma校正，因为遮罩需要保留所有纹理的信息以便能顺利达到遮挡像素的目的
* ue4中的物件至少有一个material id，也可以有多个，数量越多渲染所消耗的资源也越多，一些不起眼的小物件和大面积的物体如地板、墙壁等最好只用一个
* 一张矩形纹理所包含的图像信息可能没有完全覆盖所有区域，ue4会在解析之后丢弃那些无用的部分，为了提高效率，我们可以为纹理做适当的“裁剪”，多提供一些顶点让其一开始就忽略那些没有信息的区域，这样比每帧去分析并抛弃效率要高（称为overdraw limit）

## Blueprint

类似unity中的prefab，场景中的actor以及其组件生成蓝图后，可在场景中按该蓝图方便的生成实例，并在修改蓝图后将场景中的实例与当前蓝图的状态同步。不同的是蓝图中可以编辑事件图（event graph），它可以将游戏中原来用脚本实现的逻辑可视化成一个个节点和事件，让我们在不敲一行代码的情况下完成游戏逻辑，这东西挺牛逼的，如果做到极致，以后将不需要只处理基本业务逻辑的程序员，策划自己就能完成功能。

一个完成的蓝图可以被当成一个类，在另一个蓝图中使用，用过设置蓝图中variable的属性，可以决定某些变量是否在被其他蓝图引用时设置，或是外部是否可以直接设置变量。

蓝图分为关卡蓝图（level blueprint）和蓝图类（blueprint class），蓝图接口（blueprint interface）

UE4中可以override某些接口，而一些接口在C++中定义为const，因此其中只允许调用const函数，在蓝图中定义自己函数的时候要勾选const，否则会编译不过。

unlua相当于一个使用lua对umg进行操作的插件，优点是效率高、代码符合程序员直觉，逻辑复杂时不用担心太乱。缺点是不能打断点调试。

## 反射机制

很多语言如Java、lua都支持反射，Unlua就是使用反射实现的，具体还需要时间专门梳理

## Unlua

unlua调用引擎：直接调用有UFUNCTOIN的方法和有UPROPERTY的成员。

引擎调用unlua：在引擎中创建蓝图后，生成unlua模板，编写脚本实现ui逻辑，蓝图中还要实现一个unlua接口，并注册蓝图的路径。

TArray的lua接口定义在LuaLib_Array.cpp中，有常用的Get、Set、Length等接口。类似的，TMap的lua接口在LuaLib_Map.cpp中，常用的有Find、Add、Remove等。

lua层调用c++方法时，只要类型对了不管是指针还是对象传过去都可以，中间应该做了处理。

## 蓝图间通信

如果A蓝图可以拿到B蓝图实例，则可以在B蓝图中定义一个EventDispatcher，在需要通知的时候call，然后在A蓝图中bind相应的函数去执行触发的逻辑。

如果互相不好拿实例，那么只能在pc中定义delegate，一个Broadcast，一个绑定监听回调。

## Delegate

UE4中的delegate通过各种不同参数的宏实现

## 网络复制

## DefaultObject



## WorldContext

游戏中形形色色的Actor和其上的Component存在于Level中的Actors数组中，编辑关卡时的WorldSettings其实是该Level的设置，还有关卡蓝图ALevelScriptActor，它们都是个Actor。World中又储存了Levels数组（一个PersistentLevel和若干SubLevel），同一时间只能存在一个World，当需要切换World时，就要用到WorldContext中保存的信息，即上下文，它是管理多个World的工具。

## GameInstance

在整个Gameplay流程中存在的对象，保存了包括WorldContext的所有游戏信息，不会随着Level切换而丢失。在这之上便是引擎UEngine类，它的实例保存为一个全局的变量`GEngine`。

## Subsystems

UE中的Subsystem有Editor、Engine和GameInstance的，比较常用的是最后一个，通过继承`UGameInstanceSubsystem`，我们可以定义自己的Subsystem类，它会在GameInstance创建的时候创建，并自动初始化，当GameInstance关闭（shutdown）的时候自动析构，这样我们就用不着操心它的生命周期，在里面写需要的接口就行了。

## Actor

游戏中的基本物件，继承自UObject，有容纳Component的能力，而且可以实现层级嵌套。另外一个重要的特性是网络复制，可以将各种属性从服务器同步到客户端。

## Component

Component是用来编写功能逻辑的组件，继承自UObject，一般是比较通用的**功能**，如移动、按键输入、摄像机转动等，我们要区分在Component上实现的功能和游戏的业务逻辑，业务逻辑与游戏的玩法和表现关系紧密，而且不同游戏的差别可能很大，这些逻辑应该尽量避免放在Component上。

### MovementComponent

### InputComponent

## Pawn、Character

Pawn派生自Actor，是所有可以被玩家或AI的Controller所控制（possessed）的基类，它主要增加了三个特性：可以被Controller控制、游戏中的视觉表现和物立碰撞、可以移动。这些在引擎提供的DefaultPawn中都有体现，它增加了CollisionComponent、MovementComponent以及StaticMeshComponent。

Character派生自Pawn，可以理解为人形的可以走动的Pawn，它增加了CapsuleComponent、CharacterMovementComponent和SkeletalMeshComponent。

## Controller

Controller是一个没有实体的Actor，用来持有并控制一个Pawn的行为，由于UE设计之初便是为FPS游戏服务，因此Controller一次只能控制一个Pawn，无法控制多个，因此对于RTS等游戏类型来说不太友好。但这样做的好处是Controller可以随时拿到持有的Pawn，Pawn也可以很容易Get到当前的Controller。Controller是可以有位置信息的，这样可以让它在游戏中跟随Pawn移动，方便随时Respawn。

Controller的职责是控制Pawn，放的是关于“指挥”Pawn行动的逻辑，至于Pawn自己具体的一些行为表现，应该放在Pawn里面实现。

Pawn随时可能被销毁，有一些在关卡中需要存续的状态和数据，可以放在Controller中，具体来说就是PlayerState中，它派生自`AInfo`，是一个专门储存玩家数据的类，如玩家id、玩家名，每个玩家都会有一个PlayerState，Controller存了一个它的指针。

## GameMode、GameState

GameMode是用来控制游戏玩法和基本规则的Actor，GameMode之于World就像PlayerController之于玩家，[官方文档](https://docs.unrealengine.com/4.26/en-US/InteractiveExperiences/Framework/GameMode/)上列出了这么几项：
* 玩家和观战者数量
* 玩家如何开始游戏
* 游戏是否能暂停，暂停如何处理
* 关卡间的切换、过渡

哪些逻辑应该放在GameMode而不放在Level？前面说了GameMode是控制游戏玩法的，那么它应该关心有关玩法的逻辑，如胜利条件。对于限于每个Level自身表现的逻辑肯定是放在Level中，而对于此玩法（Mode）来说通用的逻辑可以放在GameMode中。
GameplayStatics中有GetGameMode接口，但在客户端调是拿不到的，GameMode只存在于服务器中，不会同步给任何客户端，因此如果关卡是服务器拉玩家进去的就拿不到GameMode。

如果有一些信息和事件需要同步给所有玩家，就需要通过GameState，它会和GameMode一同创建，包括游戏运行的时间、当前的GameMode、游戏是否已经开始等，和PlayerState类似，它也继承自AInfo。

## 参考

[《InsideUE4》（四）](https://zhuanlan.zhihu.com/p/23321666?refer=insideue4)及后续文章。
