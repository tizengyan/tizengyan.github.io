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
