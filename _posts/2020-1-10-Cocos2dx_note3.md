---
layout: post
title:  "Cocos2dx开发笔记3"
categories: 游戏开发
tags: Cocos2dx
excerpt: zt业务逻辑
author: Tizeng
---

* content
{:toc}

此笔记以Mac系统为平台。

## 服务器通信

以背包数据为例，`obj_PackData`方法接收背包的初始化消息，收到消息的过程为，Callback.lua脚本中处理服务器发来的消息，先分发给`DataManeger`，然后在分发给`UIManager`，调用的是各自的`dispatchMsg`方法，传入消息名称（`obj_PackData`）和数据包。`DataManager`在初始化时会包含所有处理消息的脚本名以及它们的实例，然后以传入的消息名索引对应的方法，缓存在本地，然后调用。`UIManager`的实现有所不同，每创建一个panel，`UIManager`都会按该panel的名字将其储存进一个表，分发时遍历所有已储存的panel并以传进的函数名索引查找该函数，如果找到则调用。

之前的bbl项目中若要接收服务器消息需要在脚本开头先注册消息名称，是因为在该项目中`DataManager`在分发消息前按接收消息的名称储存了该脚本的实例，然后分发时用消息名称索引需要接收该消息的实例，调用相应的方法，这样做的好处是不用每次收到消息时都去遍历有哪些数据脚本需要该消息，提升性能。`UIManager`也是类似，为了不在每次收到消息时都去遍历所有ui，也按照消息的名称来储存所有需要该消息的实例。在关闭ui时，释放空间的操作是在`UIManager`中的update方法中，目的是在下一帧才执行我们的释放操作，这样可以防止在同一帧内发生多次创建和删除操作。

## 配置读取

`g_DataManager`中的`getXmlData`和`getExeclData`负责读取xml和excel配置。

## 上漂提示

大部分上漂和提示在`ChatData`中的`showGameMsgTips`方法实现，根据传入不同的提示类型调用不同的方法去显示。

## zt任务系统


## 寻路

寻路实现在`GodConfig`脚本中，其中`goto`方法是通过`pathID`读取路线中NPC等配置，通过一系列的条件判断，最后在回调函数中调用`moveTo`方法开启寻路。注意如果在调用事件(addCallEvent)中使用了多于一个的事件，那除第一个事件的其他事件都要继续使用`add***Event`方法加入事件处理队列。
如果在副本中要先去pathID对应的一个坐标处，再调用一次`goto`或`moveTo`去最终的目标点。如果目标国家不是当前国家，就要跨国寻路，跨国寻路时要先寻路到*边境*，寻路接口在`TransportData`脚本中，如果返回为`true`则说明寻路成功，调用`gotoTransport`方法去到目标点（存疑)。如果目标点就在当前地图，当与目标点距离比期望值大时，调用主角脚本的`addMoveEvent`方法。
如果是同一个国家的不同地图，就要考虑地图之间传送点的问题，`TransportData`中的方法`findPath`就是搜索不同地图间的传送点，算法就是简单的广度优先搜索，得到从当前地图到达目标地图所需的最少*传送点*的列表，然后去列表中第一个传送点的位置，即递归的调用`goto`或`moveTo`，知道到达目标点位置。

主角脚本中有一个成员`mInputCommandQueue`用来储存输入的事件，然后每帧会被`processInput`方法处理，MoveEvent会被`on_MoveEvent`方法处理，最后调用的也是`god.moveTo`方法。

角色本身的移动逻辑是在自身脚本中的`moveTo`方法中实现，这个方法继承自`SceneNpc`脚本，最后调用的是c++中的`cCharacterExt.requestPath`方法，经过一系列传递后在CAStar.cpp中的`FindPath`方法实现寻路，实现是基本的A*寻路，用map作为开闭表记录搜索的节点，通过计算节点xy坐标的哈希值来索引。寻路完成后，路径会保存在成员`m_Path`中，然后每次查询下一个节点，就会把该节点储存到成员`m_PathPreNode`中，再用专门的方法去访问它的xy坐标，
