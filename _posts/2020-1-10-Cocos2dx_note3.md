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

## 注册回调函数

首先看`handler`和`registerClockHandler`方法的实现：

```lua
function handler(obj, method)
    return function (...)
        return method(obj, ...)
    end
end

-- 类A中实现的注册方法
function M:registerClickHandler(obj, func, ...)
    if type(func) == "function" then 
        self.m_clickCallbackArgs = {...}
        self.m_clickCallbackFunc = handler(obj, func)
    else
        self.m_clickCallbackArgs = nil
        self.m_clickCallbackFunc = nil
    end
end

-- 类A中使用回调
function M:onClick(touch, event)
    if self.m_clickCallbackFunc then 
        self.m_clickCallbackFunc(self, unpack(self.m_clickCallbackArg))
    end
end

-- 在另一个类B中生成这个类的实例instance，然后注册类B中的方法为类A实例的回调
-- instance:registerClickHandler(selfB, selfB.someFunction)
```

外部生成实例后若想注册点击回调函数，则直接向上面一样调用`registerClickHandler`方法，也就是说这里想把类B的一个方法注册为类A实例的回调函数，至于该回调在何时调用我们不关心，注册时传入的self是类B的self，进入`registerClickHandler`后`hanlder`方法会将其作为obj参数传入，并作为第一个参数传入`self.someFunction`，注意这里`handler`返回的并不是`self.someFunction`这个方法本身，而是匿名函数：

```lua
function(...) 
    return instanceB.someFunction(selfB, ...)
end
```

因此最后调用类A的方法`onCLick`时，其中作为参数传入的self是类A的实例，就算后面的参数列表为`nil`，它也会作为唯一的参数传入`self.someFunction(selfB, selfA)`。

## 创建UI

创建ui在`UIManager`中有两个接口，一个是`getOrCreatePanel`，它会先调用`getPanel`去取，如果取不到则调用用`createPanel`方法创建ui并返回，`UIManager`会储存每个创建的实例（创建是调用的`Node`节点中的`create`方法）。而如果是用`createPanelOnly`创建的ui，就不能用`getOrCreatePanel`方法来获得实例，因为它没有在`UIManager`中保存已创建ui的实例，直接调用会导致后者再以自己的方式创建并储存该ui，一般来说调用了`createPanelOnly`的类中会自己储存ui的实例，然后实现相应的get方法去取，我们应该去调用这些get方法。

有一类ui的创建较为特殊，道具点击后弹出的小弹窗，根据道具种类不同，弹窗上的按钮种类可能有区别，也可能需要不同的回调函数，创建背包界面时面对茫茫多的道具，显然应该封装一个方法来处理不同道具的callback，背包中首先为每个道具创建空的`UIItemIconBg`实例，然后根据道具data中的grid格子坐标获取itemData，然后用itemData去初始化创建的格子，其中便包括给其注册正确的callback函数，通常情况下（没有需要对比的其他道具），这个callback先创建`UIBaseTips`，它是tips的基本背景图（后面也会根据道具的data切换），然后根据该道具的baseid得到需要显示的按钮btnList，tips中按钮也需要回调，所有这些回调函数统一定义在`UITipsConfig`中，按类型进行索引，然后给当前物品弹窗tips设置正确的回调。

### 衣橱系统新功能（2020.2.19）

在衣橱ui中加入一个“进阶”页签，让衣品达到十段的外套可以继续升钻，点击提升按钮后弹出可选择提升材料的窗口，选择后点击确定消耗材料，提升外套经验，达到当前等级要求的经验后升一颗钻，按白蓝黄绿紫的顺序，每种颜色有五级，升满后变为下一个颜色，直到紫5。

这本身是一个很基础的功能，但却比预期的时间晚了24个小时才完成，主要的原因有二，一是上面总结的get和创建ui的方法使用不当，导致get到了错误的ui实例，界面收到服务器消息后没有实时刷新；二是没有很快理解`UIWardrobChoose`这个ui的创建逻辑，把自己绕进去了，浪费了很多时间。

## 寻路

寻路实现在`GodConfig`脚本中，其中`goto`方法是通过`pathID`读取路线中NPC等配置，通过一系列的条件判断，最后在回调函数中调用`moveTo`方法开启寻路。注意如果在调用事件(addCallEvent)中使用了多于一个的事件，那除第一个事件的其他事件都要继续使用`add***Event`方法加入事件处理队列。
如果在副本中要先去pathID对应的一个坐标处，再调用一次`goto`或`moveTo`去最终的目标点。如果目标国家不是当前国家，就要跨国寻路，跨国寻路时要先寻路到*边境*，寻路接口在`TransportData`脚本中，如果返回为`true`则说明寻路成功，调用`gotoTransport`方法去到目标点（存疑)。如果目标点就在当前地图，当与目标点距离比期望值大时，调用主角脚本的`addMoveEvent`方法。
如果是同一个国家的不同地图，就要考虑地图之间传送点的问题，`TransportData`中的方法`findPath`就是搜索不同地图间的传送点，算法就是简单的广度优先搜索，得到从当前地图到达目标地图所需的最少*传送点*的列表，然后去列表中第一个传送点的位置，即递归的调用`goto`或`moveTo`，知道到达目标点位置。

主角脚本中有一个成员`mInputCommandQueue`用来储存输入的事件，然后每帧会被`processInput`方法处理，MoveEvent会被`on_MoveEvent`方法处理，最后调用的也是`god.moveTo`方法。

角色本身的移动逻辑是在自身脚本中的`moveTo`方法中实现，这个方法继承自`SceneNpc`脚本，最后调用的是c++中的`cCharacterExt.requestPath`方法，经过一系列传递后在CAStar.cpp中的`FindPath`方法实现寻路，实现是基本的A*寻路，用map作为开闭表记录搜索的节点，通过计算节点xy坐标的哈希值来索引。寻路完成后，路径会保存在成员`m_Path`中，然后每次查询下一个节点，就会把该节点储存到成员`m_PathPreNode`中，再用专门的方法去访问它的xy坐标，

## zt任务系统


