---
layout: post
title: "UE4笔记3——网络同步"
date: 2022-06-30
categories: 引擎
tags: ue4
excerpt: 尝试在ue4中开发技能编辑器
author: Tizeng
---

* content
{:toc}


## 地图切换（[官方文档](https://docs.unrealengine.com/4.27/zh-CN/InteractiveExperiences/Networking/Travelling/)）

多人游戏时地图加载有两种模式，一种是无缝（seamless），一种是非无缝（non—seamless），它们的主要区别在于无缝是非阻塞的（non-blocking），而非无缝是阻塞的操作（blocking）。非无缝切换时客户端会先断开连接，重连后再加载新的地图，有三种情况必然会发现非无缝转移：初次加载、客户端初次连接服务器、重新开始游戏时。

无缝切换时会有一个过度地图（transition map），我们可以将当前场景中的部分Actor带进最终地图中，通过重载GameMode或PlayerController中的`GetSeamlessTravelActorList`方法，把想要直接带入新关卡的Actor加到一个列表中。

## 客户端连接DS（2022.6.30）

客户端在连接ds时，会经过如下几个阶段：

1. 向ds请求，得到许可后开始加载地图（c->s: Hello, s->c: Challenge, c->s: Login）
2. ds调用`AGameModeBase::PreLogin`，如果没有问题就通知客户端（s->c: Welcome）
3. 客户端设置`bSuccessfullyConnected=true`，在`UEngine::LoadMap`中加载地图（同时发送NetSpeed），加载完毕后告知ds（c->s: Join）
4. ds调用`AGameModeBase::Login`创建PlayerController并同步到客户端
5. 最后ds调用`AGameModeBase::PostLogin`，此时ds可以安全的调用PlayerController上的RPC函数，且在`HandleStartingNewPlayer`中初始化Player

服务器在`UWorld::NotifyControlMessage`中处理客户端的消息，客户端则在`UPendingNetGame::NotifyControlMessage`等方法中处理。

`AGameMode`中处理了多人射击游戏相关的内容，它维护了一个`MatchState`用来表示当前比赛的状态，当其进入InProgress状态时，说明游戏正式开始，`HandleMatchHasStarted`会被调用，通知所有Actor调用BeginPlay。

## 网络连接（Connection）

- UPlayer：包含一个PlayerController，代表该玩家
- ULocalPlayer：继承自`UPlayer`，客户端上每个玩家都有一个LocalPlayer，增加了Viewport有关的信息。服务器上可能不存在
- UNetConnection：继承自`UPlayer`，包含NetDriver和Channel
- UNetDriver：一个ServerConnection（客户端情况），多个ClientConnections（服务器情况），服务器上ServerConnection会为空
- UChannel：
- UActorChannel：继承自`UChannel`，

- FNetworkGUID：用于在网络同步中识别对应的UObject

`APlayerController`中包含与之关联的Player（`UPlayer`，可能是LocalPlayer也可能是NetConnection），同时还有单独的NetConnection成员。

客户端自己创建的Actor，Role是Authority，RemoteRole会是None，就算将其Owner设置为有连接的PlayerController，它对服务器来说也没有意义。

### 时序问题

参考[知乎文章](https://zhuanlan.zhihu.com/p/34721113)。
如果RPC函数和Actor同步在服务器上同时发生，理论上RPC会较快到达客户端，但是由于网络延迟等问题这是不确定的，比如服务器上修改了Actor上的某个变量，我们期望客户端上收到同步后调用绑定的OnRep函数，如果此时有RPC函数也修改了客户端上的值，那么这个函数可能就不会被调用了。
还有一种情况是用Controller去Possess一个新生成的Pawn时，会调用PRC函数`ClientRestart`到客户端，这时该Pawn可能还没有同步下来，导致客户端拿不到。引擎中对这种情况做了处理，PlayerController的`PlayerTick`中会时刻检查客户端上的Pawn是否为空，如果是则持续尝试GetPawn，直到成功拿到并Set，再去调用`ClientRestart`。

RPC与属性同步还有一个执行时机的区别，如果服务端调用Client函数时客户端由于距离等问题还没有这个Actor，那么客户端之后便再也不会收到，此时就可以用属性同步来解决，只要客户端和服务端的数据不一致便会触发同步。

### Relevance


