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

- Packet
- `FNetworkGUID`：用于在网络同步中识别对应的UObject，用一个uint32来区分

- `FObjectReplicator`：用来表示正在被复制的对象或者执行的RPC，包含该对象的GUID

`APlayerController`会在客户端连接ds时被创建，并绑定对应的connection，每个连接至ds的客户端都被看做是一个connection。
PC中包含与之关联的Player（`UPlayer`，可能是LocalPlayer也可能是NetConnection），同时还有单独的NetConnection成员。
这也说明只有和玩家相关的actor才有连接。

当连接被关闭时，NetDriver的`TickDispatch`中会检测到，然后调用对应连接的`CleanUp`方法清除数据，最终这个连接所属的Controller会调用`OnNetCleanup`将自己释放。

### Owner（[官方文档](https://dev.epicgames.com/documentation/en-us/unreal-engine/actor-owner-and-owning-connection-in-unreal-engine)）

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

## RPC

ProcessEvent
CallRemoteFunction
UNetDriver::ProcessRemoteFunction
AActor::GetFunctionCallspace
ReplicateSubobjects

## 属性同步（Replicated）

SendBuffer
GetLifetimeReplicatedProps
DOREPLIFETIME_CONDITION_NOTIFY做了什么

## 预测

移动组件上有关于预测的逻辑
GAS中也实现了一套预测回滚机制，这个放在单独的地方讨论。
