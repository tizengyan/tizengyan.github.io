---
layout: post
title: "UE源码学习——网络同步"
date: 2022-06-30
categories: UE源码学习
tags: 网络
excerpt: 总结一下网络同步相关的问题
author: Tizeng
---

* content
{:toc}


## 客户端连接DS（2022.6.30）

客户端在连接ds时，会经过如下几个阶段：

1. 向ds请求，得到许可后开始加载地图（c->s: Hello, s->c: Challenge, c->s: Login）
2. ds调用`AGameModeBase::PreLogin`，如果没有问题就通知客户端（s->c: Welcome）
3. 客户端设置`bSuccessfullyConnected=true`，在`UEngine::LoadMap`中加载地图（同时发送NetSpeed），加载完毕后告知ds（c->s: Join）
4. ds调用`AGameModeBase::Login`创建PlayerController并同步到客户端
5. 最后ds调用`AGameModeBase::PostLogin`，此时ds可以安全的调用PlayerController上的RPC函数，且在`HandleStartingNewPlayer`中初始化Player

服务器在`UWorld::NotifyControlMessage`中处理客户端的消息，客户端则在`UPendingNetGame::NotifyControlMessage`等方法中处理。

`AGameMode`中处理了多人射击游戏相关的内容，它维护了一个`MatchState`用来表示当前比赛的状态，当其进入InProgress状态时，说明游戏正式开始，`HandleMatchHasStarted`会被调用，通知所有Actor调用BeginPlay。

`PostLogin`中会拿到之前生成的Controller，调用它的一个Client函数，生成AHUD，也就是说只有客户端有，它的`RemoteRole`是NONE。

### 地图切换（[官方文档](https://docs.unrealengine.com/4.27/zh-CN/InteractiveExperiences/Networking/Travelling/)）

多人游戏时地图加载有两种模式，一种是无缝（seamless），一种是非无缝（non—seamless），它们的主要区别在于无缝是非阻塞的（non-blocking），而非无缝是阻塞的操作（blocking）。非无缝切换时客户端会先断开连接，重连后再加载新的地图，有三种情况必然会发现非无缝转移：初次加载、客户端初次连接服务器、重新开始游戏时。

无缝切换时会有一个过度地图（transition map），我们可以将当前场景中的部分Actor带进最终地图中，通过重载GameMode或PlayerController中的`GetSeamlessTravelActorList`方法，把想要直接带入新关卡的Actor加到一个列表中。

## 网络连接（Connection）

先看一下相关的基本类型：

- `UPlayer`：包含一个PlayerController，代表该玩家
- `ULocalPlayer`：继承自`UPlayer`，客户端上每个玩家都有一个LocalPlayer，增加了Viewport有关的信息。服务器上可能不存在
- `UNetConnection`：继承自`UPlayer`，包含NetDriver的引用。有多个Channel，并用一个Map对应了名字，可以按名字查找
- `UNetDriver`：保存在当前的World中，包含一个ServerConnection（客户端情况），多个ClientConnections（服务器情况），服务器上ServerConnection会为空

- `UChannel`：包含一个OwnerConnection。用FName存了自己的名字
- `UActorChannel`：继承自`UChannel`，用来处理Actor的同步和RPC，包含ActorNetGUID（`FNetworkGUID`），ActorReplicator（`FObjectReplicator`），每个同步的Actor都对应了一个ActorChannel
- `UControlChannel`：每个Connection中只有一个
- `UVoiceChannel`：每个Connection中只有一个

- Packet：网络传输的基本单位，可以包含多个bunch
- bunch：引擎网络同步的基本单位，分为发送用的`FOutBunch`和接收用的`FInBunch`
- `FNetworkGUID`：用于在网络同步中识别对应的UObject，用一个uint32来区分

- `FObjectReplicator`：用来表示正在被复制的对象或者执行的RPC，包含该对象的GUID

`APlayerController`会在客户端连接ds时被创建，并绑定对应的connection，每个连接至ds的客户端都被看做是一个connection。
PC中包含与之关联的Player（`UPlayer`，可能是LocalPlayer也可能是NetConnection），同时还有单独的NetConnection成员。
这也说明只有和玩家相关的actor才有连接。

当连接被关闭时，NetDriver的`TickDispatch`中会检测到，然后调用对应连接的`CleanUp`方法清除数据，最终这个连接所属的Controller会调用`OnNetCleanup`将自己释放。

### Owner

[官方文档](https://dev.epicgames.com/documentation/en-us/unreal-engine/actor-owner-and-owning-connection-in-unreal-engine)。

Owner和所属连接（owning connection）的概念在网络同步中很重要，每个actor上都有一个Owner的引用，它可能为空，当我们用一个PlayerController控制一个Pawn时，这个controller就是Owner，一个actor所属的连接和它所关联的controller有关。
RPC调用时服务器通过对应actor的所属连接，找到正确的客户端进行远程调用。

通过方法`GetNetConnection`可以获取一个actor所属的连接，它内部的实现就是再调用Owner的这个方法，递归的往上找，而Pawn则是去调用当前controller的这个方法。如果一个component需要网络复制，除了自身需要设置replicated，它的Owner也必须要设置。

客户端自己创建的Actor，Role是Authority，RemoteRole会是None，就算将其Owner设置为有连接的PlayerController，它对服务器来说也没有意义。

### 时序问题（2022.7.5）

参考[知乎文章](https://zhuanlan.zhihu.com/p/34721113)。
如果RPC函数和Actor同步在服务器上同时发生，理论上RPC会较快到达客户端，但是由于网络延迟等问题这是不确定的，比如服务器上修改了Actor上的某个变量，我们期望客户端上收到同步后调用绑定的OnRep函数，如果此时有RPC函数也修改了客户端上的值，那么这个函数可能就不会被调用了。
还有一种情况是用Controller去Possess一个新生成的Pawn时，会调用PRC函数`ClientRestart`到客户端，这时该Pawn可能还没有同步下来，导致客户端拿不到。引擎中对这种情况做了处理，PlayerController的`PlayerTick`中会时刻检查客户端上的Pawn是否为空，如果是则持续尝试GetPawn，直到成功拿到并Set，再去调用`ClientRestart`。

RPC与属性同步还有一个执行时机的区别，如果服务端调用Client函数时客户端由于距离等问题还没有这个Actor，那么客户端之后便再也不会收到，此时就可以用属性同步来解决，只要客户端和服务端的数据不一致便会触发同步。但是有些信息比如特效，我们只希望播一次，如果有客户端延迟进入了同步范围，我们希望只同步Actor的状态而不再播放一次特效，这个可以将特效用NetMulticast、其他状态用Replication来解决。

## 相关性（[Relevancy](https://dev.epicgames.com/documentation/en-us/unreal-engine/actor-relevancy-in-unreal-engine)）

场景中的actor数量可能非常多，而并不是每个actor都需要同步，为了提高效率，服务器只会同步和对应客户端有**相关性**的actor过去。
调用actor上的`IsNetRelevantFor`来查看它是否与某个连接相关，它会根据actor上配置的策略去查看Owner的同名方法或直接判断输入的viewer是否是自己的Owner，亦或是通过距离（`NetCullDistanceSquared`）判断。

ReplicationGraph也是一种相关性的优化，它主要是利用actor在场景中空间上的相关性，将场景划分为很多格子，每个actor上设置一个`CullDistance`，在这个距离内的所有格子都被视为需要同步的区域，当一个连接的viewer进入了某个格子所代表的区域时，对应的actor就需要对其同步。参考[知乎文章](https://zhuanlan.zhihu.com/p/56922476)。

### Subobject同步

[官方文档](https://dev.epicgames.com/documentation/en-us/unreal-engine/replicating-uobjects-in-unreal-engine#defaultsubobjectreplication)。

要同步UObject及其上面的属性，第一步是创建，以component为例，分为预先创建和动态创建两种，前者通过`CreateDefaultSubobject`在构造中创建，这种情况下服务器和客户端各自在spawn这个actor时执行，由于标记为`bReplicated`的actor会在spawn结束后自动同步到客户端，因此两端都会自动有。
预先创建的对象`IsNameStableForNetworking`会返回`true`。
而动态则是游戏运行时通过`NewObject`创建的，它并不会自动同步到客户端上，直到我们使用了下面的两种方式主动进行同步。

第一种是使用subobject的注册列表（`FSubObjectRegistry`）。首先在持有待同步UObject的actor上设置标记`bReplicateUsingRegisteredSubObjectList`为`true`，然后调用`AddReplicatedSubObject`将对象加入`ReplicatedSubObjects`列表，这样在actor同步时会将这个列表中的对象一并写入`FOutBunch`中。需要注意的是如果要删除同步的对象，要先将其冲列表中移除，因为列表持有的是对象的裸指针，不主动移除可能发生崩溃。

第二种是重写actor上的`ReplicateSubobjects`方法，它会传入`UActorChannel`和数据bunch，我们通过调用channel上的`ReplicateSubobject`方法将要同步的对象写入bunch中。
查看源码可以得知，`AActor`中就是使用这个方法将`ReplicatedComponents`数组中的组件进行同步的。

无论使用上面哪种方式进行同步，UObject需要实现`IsSupportedForNetworking`并返回`true`，同时也要实现`GetLifetimeReplicatedProps`并在其中用宏注册需要同步的成员。

## 属性同步原理（2025.3.15）

参考[《Exploring in UE4》网络同步原理深入](https://zhuanlan.zhihu.com/p/55596030)。

和属性同步相关的主要有以下几个类：

* `FRepLayout`：每种类型都有一个，包含这个类型需要同步的所有属性，并提供同步所需的接口，如`ReplicateProperties`和`CallRepNotifies`。
其中Parents数组（`FRepParentCmd`类型）表示所有需要同步的上层（top level）属性，Cmds数组（`FRepLayoutCmd`类型）表示上层属性所嵌套的子属性，如数组或结构中的属性（为什么要叫cmd？命令模式吗）。
在NetDriver中有一个map可以通过UClass来索引对应的layout。在初次获取时如果没找到就会创建，将对应UClass中所有需要同步的属性加到Parents数组中，如果该属性是数组或结构，则递归的执行加入到Cmds中。
* `FRepState`：每个连接的每个对象都有一个，分别储存了接收和发送需要同步的信息，以及当前同步的状态条件
    * ReceivingRepState：缓存了对象的数据（StaticBuffer），用来比较是否发生变化，以及`FProperty`数组RepNotifies用来在同步发生的时候在客户端调用OnRep函数（FProperty中有一个`RepIndex`成员，用来在上面提到的Parents中获取数据，再通过和StaticBuffer的偏移获取同步回调函数的参数信息，这里有个问题，OnRep函数应该是没有参数的）
    * SendingRepState：缓存了属性tracker，还有变化的历史
* `FRepChangedPropertyTracker`：追踪发生变化的属性以及同步的条件，判断是否需要同步
* `FObjectReplicator`：最终执行同步的类，缓存了需要同步的UObject对象指针，与StaticBuffer中的信息比较

### GetLifetimeReplicatedProps做了什么

想要使某个成员同步必须重写这个方法进行注册，`FRepLayout`初始化的时候会拿到传入UClass的CDO调用这个方法，获取到这个类中需要被同步的成员,`DOREPLIFETIME`系列的宏所作的就是将输入的成员信息（包括同步条件）注册到数组LifetimeProps中，最终被layout储存在Parents数组中。

## RPC原理

被标记为RPC的函数在`gen.cpp`中会生成一个实现，通过生成的参数结构和名字调用`ProcessEvent`，还会有一个`exec`开头的静态函数，其中先调用了`_Validate`结尾的实现，如果通过则调用`_Implementation`结尾的实现，这就是为什么我们可以只需要实现一个后者执行逻辑。

`ProcessEvent`执行时会通过`GetFunctionCallspace`判断是否需要执行RPC，如果是则调用`CallRemoteFunction`，最终通过NetDriver中的`ProcessRemoteFunction`找到对应的连接去执行。
客户端收到bunch之后，进行一系列处理和条件判断，最终也调用`ProcessEvent`到`Invoke`，执行UFunction上绑定的`exec`开头的函数。

## 可靠传输

## 预测

移动组件上有关于预测的逻辑
GAS中也实现了一套预测回滚机制，这个放在单独的地方讨论。
