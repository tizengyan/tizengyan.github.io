---
layout: post
title:  "Cocos2dx入门笔记"
categories: 游戏开发
tags: Cocos2dx
excerpt: 开始学习Cocos引擎
author: Tizeng
---

* content
{:toc}

此笔记以Mac系统为平台。

## 初始化新项目

Cocos[官方文档](https://docs.cocos.com/cocos2d-x/manual/zh/editors_and_tools/cocosCLTool.html)居然把新建项目的说明放到末尾的地方，真是有毒。

`cocos new`关键词用来创建新项目，格式形如`cocos new <game name> -p <package identifier> -l <language> -d <location>`，然后需要编译成二进制文件，格式和上面类似，形如`cocos compile -s <path to your project> -p <platform> -m <mode> -o <output directory>`，如果终端目录已经在最后创建的项目需要运行，

## i18n

指国际化与本地化（internationalization and localization）是指修改软件使之能适应目标市场的语言、地区差异以及技术需要。常被分别简称成i18n（18意味着在“internationalization”这个单字中，i和n之间有18个字母）及L10n。

## Cocos中类的实现

下面是Cocos中关于`class`这个函数的实现，它是一个创造多继承类的函数：

```lua
function class(classname, ...)
    local cls = {__cname = classname}

    local supers = {...}
    for _, super in ipairs(supers) do
        local superType = type(super)
        assert(superType == "nil" or superType == "table" or superType == "function",
            string.format("class() - create class \"%s\" with invalid super class type \"%s\"",
                classname, superType))

        if superType == "function" then
            assert(cls.__create == nil,
                string.format("class() - create class \"%s\" with more than one creating function",
                    classname));
            -- if super is function, set it to __create
            cls.__create = super
        elseif superType == "table" then
            if super[".isclass"] then
                -- super is native class
                assert(cls.__create == nil,
                    string.format("class() - create class \"%s\" with more than one creating function or native class",
                        classname));
                cls.__create = function() return super:create() end
            else
                -- super is pure lua class
                cls.__supers = cls.__supers or {}
                cls.__supers[#cls.__supers + 1] = super
                if not cls.super then
                    -- set first super pure lua class as class.super
                    cls.super = super
                end
            end
        else
            error(string.format("class() - create class \"%s\" with invalid super type",
                        classname), 0)
        end
    end

    cls.__index = cls
    if not cls.__supers or #cls.__supers == 1 then
        setmetatable(cls, {__index = cls.super})
    else
        setmetatable(cls, {__index = function(_, key)
            local supers = cls.__supers
            for i = 1, #supers do
                local super = supers[i]
                if super[key] then return super[key] end
            end
        end})
    end

    if not cls.ctor then
        -- add default constructor
        cls.ctor = function() end
    end
    cls.new = function(...)
        local instance
        if cls.__create then
            instance = cls.__create(...)
        else
            instance = {}
        end
        setmetatableindex(instance, cls)
        instance.class = cls
        instance:ctor(...)
        return instance
    end
    cls.create = function(_, ...)
        return cls.new(...)
    end

    return cls
end
```

具体到项目中，`cc.exports._class`这个表在主函数入口被初始化为一个空表，一些基类被创建完后存入其中，方便后面派生子类时调用。

`_host`存储的是组件的“宿主”，也就是这个组件目前挂在谁身上，当我们想访问时就调用`_host`，它会在组件被创建时被初始化。

找接口时先找可能需要的类定义，如果不在Lua脚本中多半就是在cpp文件中。

## 公共函数 handler

这是cocos定义在functions.lua文件中的一个函数，用来将某个对象和一个方法绑定，它的实现如下：

```lua
function handler(obj, method)
    return function(...)
        return method(obj, ...)
    end
end
```

可以看到它将参数`obj`作为方法`method`的第一个参数返回，同时它们被封装在一个匿名函数中，输入的其他参数会作为`method`方法的后续参数输入。`handler`通常被用来定义各种回调函数。

## 任务1：拾取物抛物线优化

### 掉落物品

击败场景中的敌人后会有物品掉落，这个掉落的过程是抛物线。

`update`函数接收一个参数`dt`，是当前帧距离上一帧的时间（deltaTime），为了计时，可以维护一个全局变量，每次在调用`update`时都对它进行累加，以记录过去的时间

### 掉落物品的运动

掉落的道具如果会上下浮动会比只是静止在地上体验要好，这个过程我们可以通过正弦函数来模拟，`sin`函数的波形就是一个振动的过程，根据振幅、周期不同而变化，我们可以在`update`函数中对要上下浮动的物体的y轴坐标每帧都带入`sin`函数中计算得出一个新值，那么运行后就会有该物体在原地上下浮动的效果。

要求在2D平台游戏中触碰收集物后，收集物从掉落处以抛物线形式运动到角色头顶然后落下，不论角色如何移动，收集物在运动过程中始终跟随角色，并最终接触角色头顶。

这个问题分两个部分，或者说横纵两个方向，x方向要求收集物必须在固定时间内移动到角色所在位置，y方向需要在相同的时间内完成先上后下的上抛运动，二者合一就是我们需要的效果。

要注意纵坐标向下是正方向，向上是负方向。

首先复习一下高中物理，匀加速运动、自由落体运动、斜抛运动。我们这里相当于是一个上抛运动加一个水平的匀速运动。

首先将速度分为x、y轴两个方向分开计算

三种情况：

- 起始点低于落点低于最高点
- 落点低于起始点低于最高点
- 落点低于最高点低于起始点（这种情况只会在物品还在上升的同时，角色开始下落出现）

注意lua中`and`和`or`的优先级很低。

理一下掉落的逻辑，首先怪掉落什么物品掉落多少由服务器决定，如果有掉落服务器会发送增加场景`item`的消息，由`SceneManager`的添加方法`addSceneObject`在场景中添加，这个方法会根据传入的`classid`判断应该创建哪个类的实例，再根据传入的`thisid`进行创建，如果创建成功，则将这个实例登记为每帧刷新的物体，`SceneManager`会在`update`方法中对登记的每个`SceneObject`（在这里具体为`SceneDropItem`），调用刷新方法`refreshPresentation`，而它会调用`createEntity`方法，在这个方法中调用C++的接口，读取Sprite素材，并定义渲染顺序，最后添加`SceneComponent`，调用其中的`addComponent`方法，创建掉落物品的component类对象：`SceneComponent_DropItem`，该物品如何掉落，掉落后作何表现，被拾取后作何表现，最后如何删除自己都定义在这个类的脚本中。

## 任务2：给字体加入描边颜色配置

很简单的一个需求，但是找API找半天，原来是该方法是由编译器生成后存在一个lua文件中，然后手写的lua脚本在调用时更换了对象的名字，所以一直找不到。

## 任务3：给任务称号文字和颜色配置

看似简单的需求涉及到很多复杂的关系

表中索引替换

函数参数不对

总结一下对象、函数调用顺序和思路，

实现后只有自己能看到称号，但是需要场景中所有其他玩家都看到互相的称号，这就需要服务器那边发信息过来，告诉客户端场景中哪一个角色的头衔发生了变更，然后将变更后的头衔信息储存在`SceneCharacter`对象中，注意不是存在`SceneMainCharacter`对象中，否则这样玩家就只能看见自己头上有头衔，别的玩家看不见，我们也看不见其他玩家的头衔。

称号字体的颜色是根据excel表格读出的文件`title.lua`中的配置决定的。

### 富文本解析（Rich Text Format）

居然可以带图片

## 任务4：上漂效果优化（2019.10.02）

cocos2dx中的sequence应用，富文本载文字加图片，配置上漂距离和渐隐时间

理一下这个功能涉及到类和对象的关系，首先有一个消息通知基类`UINoticeBase`（派生自`UIPanel`），它定义了一个处理消息的队列。基础上漂`UINoticeCommon`继承自`UINoticeBase`，它定义了消息如何在屏幕中显示，即我们需要的上漂效果，类`NoticeManager`用来根据外部传入的参数，决定调用哪个消息类，如`UINoticeCommon`或其他处理消息类中的方法。最后我们看到在`PackData`中用对象`g_NoticeManager`调用了拾取道具的效果，在`MainData`中同样用它调用了拾取金币的效果。

每次更新`proto`文件后要make protoPb更新协议，否则会解码失败。

对象完成功能后计时空闲时间，如果时间超出还没有被使用，则主动销毁。

队列（或栈）中的每条消息需要延迟显示，避免在屏幕上的重叠，只需要当容器不为空时在`update`方法中计时，在计时器大于零时累加，超过阈值后显示消息，并将计时器置零。但是这样有一个问题，尽管容器中的消息会按照我们设定好的阈值依次输出，但是被存入的第一条消息也被延迟了，因此玩家每次看到消息时其实整体也被延时了相应的时段，要解决这个问题我之前以为只要在阈值判断处加一个判断变量，判断容器是否为空，如果为空则说明是第一条消息，那么直接显示不进行延时，但是这样的话容器其实就会永远为空，因为根本不会有第二条消息进入容器，所有消息刚进去马上就被弹出，解决方法其实很简单，我们保留之前的设计，只是在第一条消息存入时多存入一条消息占位，这样后面的消息就可以进入容器了，最后在弹出消息时不要忘记检查容器的大小，如果为1则说明剩下的是刚开始压入的占位消息，我们直接将其弹出即可。

特殊上漂要更换背景图，为了达到这个操作，以前的代码需要重构，加入新类`UINoticeLayer`管理一连串的上漂，现在计时和消息容器操作都在这个类中。

在cocos studio的图形界面中配置好的UI界面（或其他东西），可以导出成csd文件供程序使用，cocos会自动将csd文件转化为一个同名的`.lua`文件，通过这个文件我们可以了解到该UI的配置和层级关系，我们再在lua脚本中通过特定的方法与之绑定，并将其中的“物体”名映射到我们希望的对象成员名称，就可以在该脚本中获取到该成员了。

背包还分为普通背包和时装背包，获得时装后先加入普通背包，点击使用后进入时装背包，但实际上进入时装背包的是另一个道具，只是看起来一样，代码层面上，我们需要维护一个根据`itemid`来得到应进入背包类型的表，每次获得道具后都更新这个表，这样不论有几种类型的背包，我们在通过`itemid`来获取道具时都能根据这个表去正确的背包中查找。

### 防止重复消息堆积

除了拾取道具，还有一种消息也需要上漂，就是系统提示，如按下攻击按钮，但是附近没有敌人，会显示一个“目标为空”的上漂，如果不做处理，在没有敌人的情况下一直按攻击按钮，消息就会源源不断的出现在屏幕上。为了防止这种情况出现，我们新建一个上漂类型`Enum.NoticeType.NoticeCommonUnique`，以这个类型传入的消息会先调用`isNoticeCommonUnique()`函数进行判断，如果返回为`true`才将消息推入容器，这个判断函数首先会检查当前容器中是否存在相同字符，如果有直接返回`false`，如果没有，再检查所有已被激活的panel中有没有panel含有该字符，如果有返回`false`，如果没有返回`true`。

注意我们只对需要去重的消息做这个处理，其他情况如连续拾取同样的物品并不会受影响。

## 任务5：背包一键丢弃功能（2019.10.08）

需求是实现一个可以一键丢弃某个装备品质以下的所有装备。首先需要一个确认弹窗，要创建一个新的弹窗我们需要新建一个类，继承自`UIPanel`，同时我们需要使用在cocos studio中建立的模板`UICommBox`，根据上面学到的，我们将这个新类与该lua文件绑定，映射物体名称到成员变量名，就可以修改模板属性来配置弹窗了。弹窗无非三个属性需要配置，描述、确认按钮、取消按钮，其中配置描述最简单，直接读取相应配置文件中的字段设置即可。然后是取消按钮，直接实现一个关闭弹窗的方法，并将方法名与取消按钮绑定即可。最后是确认按钮，我们需要在确认键按下后执行一个回调函数，
需求是实现一个可以一键丢弃某个装备品质以下的所有装备。首先需要一个确认弹窗，要创建一个新的弹窗我们需要新建一个类，继承自`UIPanel`，同时我们需要使用在cocos studio中建立的模板`UICommBox`，根据上面学到的，我们将这个新类与该lua文件绑定，映射物体名称到成员变量名，就可以修改模板属性来配置弹窗了。弹窗无非三个属性需要配置，描述、确认按钮、取消按钮，其中配置描述最简单，直接读取相应配置文件中的字段设置即可。然后是取消按钮，直接实现一个关闭弹窗的方法，并将方法名与取消按钮绑定即可。最后是确认按钮，我们需要在确认键按下后执行一个回调函数，

下一步是找到需要激活这个弹窗的地点，我们的例子中是背包的装备界面，按钮的位置已经配置好，同样是在csd文件中，因此我们要在背包界面的对应lua文件中找到相应按钮的名字，实现它的方法`onTouch`，设置在按下按钮并放开时创建弹窗UI并显示，剩下的就交给弹窗的逻辑了。

### 优化

绑定按钮功能的方法需要更新。

弹窗不会在同一时间累加，也就是说不会同时出现多个弹窗，因此为每个弹窗都创建一个新类是不必要的，好的设计是在需要弹窗的时候动态的设置，这样我们就只需要一个类。
这个类应该是一个通用弹窗类型，需要弹窗的时候用一个`set`方法将描述、回调函数传入，同时可能需要设置取消按钮和确认按钮以及按下取消后可能需要的回调函数，为了增加通用性，将外部传入的这些参数归纳进一个table，在`set`方法中去读这个表，如果相应的索引不为`nil`，则更新相应的属性。之前困扰我的关于回调函数的参数问题，其实可以被lua的闭包特性很好的解决。

总的逻辑就是，背包界面的“一键删除”按钮呼出弹窗，用`UIManager`中定义的方法`getOrCreatePanel`并输入弹窗类型名称创建，然后用之前定义方法对其设置，这个弹窗的确认回调函数是给服务器发一条要求删除装备消息的方法，这个方法定义在`PackData`中，它遍历当前背包中的所有装备，将品质低于蓝色的装备的`thisid`存入一个table，最后将这个table封装入prototype消息发给服务器。

## 任务6：buff倒计时显示优化（2019.10.17）

将任务buff在UI上显示的剩余时间的格式变更为只显示当前剩余时间的最大单位，如一分钟以上一小时一下只显示分钟数，一小时以上一天以下只显示小时，一天以上只显示天。从服务器接收到更新buff的消息后（`StateData`），用`UIManager`的方法`getPanel`调用`UIMainOperation`的实例，然后再用该实例根据场景中的对象类型，调用相应显示该类型BuffList的方法，然后绑定在其上的`UIBuffCell`对象就会在`updateTime`方法中计时，其中计时的逻辑是服务器发过来一个以整数表示的时间（从1970年1月1日开始的秒数），然后在`updateTime`中每次去和当前的时间相减，这个当前时间也要从服务器那里获得，`Util`中有专门的方法`getServerTime`，它会得到一个当前的时间，当它们的差值小于零时，就表示时间到了。至于时间结束后人物身上的buff如何消失，属于状态转移的内容。

## 任务7：增加获得称号上漂

这个功能很容易，有前面的上漂功能做基础，一下就加了，只是现在发现获得货币、道具和称号的上漂是分开实现的，各自在不同的地方，而且还在不同的Data脚本，现在的任务变成了将上漂消息逻辑重构，消息统一在`MainData`中处理上漂消息“UserPopTipsInfo”。

## 聊天界面

功能为屏幕右下角可以看到小聊天框，其中包含系统、当前、世界三种类型的消息，点击之后弹出聊天主界面，可以选择在世界和当前两个频道发言，发言方式为点击发送框，输入消息，点击发送按钮，发送后可以在聊天界面看到发出的消息，同时角色头顶也会出现一个短暂存在的消息气泡，显示刚才的消息，方便其他玩家查看。不同的频道中收到的消息不同，之前发出的消息会保存在相应的频道中，呼出聊天界面即可查看。



## 背包界面

点击屏幕上的背包图标，出现背包界面，装备和物品在同一个界面，时装在另一个界面，由一个按钮进行切换，两个界面都由若干个格子构成，每种类型的道具或某个装备占一个格子，格子正中间显示物品装备icon，背景颜色根据品质决定，点击后出现小窗显示详细讯息和两个按钮，分别是穿戴和丢弃。

实现上，主界面的背包图标绑定按钮在`UIMainOperation`中，按下后调用`UIManager`的`getOrCreatePanel`方法得到`UIPack`对象，然后调用`ViewBase`中的`ShowToScene`方法将其显示在屏幕上。

背包的逻辑在`UIPack`中，它的`onCreate`方法（也就是创建时）调用了一个初始化前面提到的格子的方法`initTabBtnTableView`，它会创建一个`ccui.Layout`，名为`subPanel`。创建装备栏和时装栏的过程类似，这里只总结一下装备栏的创建，首先调用`createPackTableView`方法，它根据`PackData`中的数据得到横纵两个方向的格子数量，并定义一个局部函数`itemCreateAndResetFunc`，它会根据传入的参数判断是否创建一个新的`UIItem`，或对传入的进行调整，然后生成（new）一个`UIGridTableView`的实例，并调用初始化方法`initTableView(parentLayout, itemCreateAndResetFunc)`，它创建`cc.TableView`，大小为根据创建实例时输入的格子数量和间隔像素计算出的长宽（单位为像素），然后将其加入`parentLayout`也就是`subPanel`的子节点中。

`itemCreateAndResetFunc`方法尽管定义在`UIPack`中的一个成员方法中，但它被传入tableview中时会被保存在该`UIGridTableView`对象中，用来创建每一个格子。

### 背包整理

整理按钮按下后背包中的物品按类型和品质排好序后依次从左到右显示在背包的格子中。

整理按钮被按下后，其绑定的函数会调用`PackData`中的方法`reqArrangeItem(curPackType)`请求整理当前类型的背包，在整理之前要调用`reqMergePackData`方法先合并一样的道具，合并的逻辑是，根据背包中物品的thisid遍历所有物品，然后用baseid判断其是不是同一件物品，并对其数量进行累计，注意这里只会记录数量小于最大数量上限的物品，然后将它们存入一个表`waitMergeList`，物品的baseid作为键，值为一个储存了所有拥有该baseid物品的信息以及总数的表`sameBaseIDList`（这个表出了储存baseid相同的物品信息外，还储存了它们当前的总个数`sumNum`以及每格格子所允许存放的最大个数`singleMaxNum`），合并时对`waitMergeList`进行遍历，对其中的每一个表`sameBaseIDList`，进行合并操作，注意在合并前，需检查合并完成后所需的格子数量是否小于或等于原先的数量，即理论上所需的总格子数（通过计算物品总数和该物品每格可放最大数量的比值得到）需小于等于现有的物品所占格子数（即`#sameBaseIDList`）。

具体到每种道具的合并由函数`merge(itemList, gridNum)`完成，这里传入的list就是某个单个道具的表，我们需要对其进行合并，方法是维护一前一后两个下标指针，分别是合并的目标格子`dest`，和从后面开始扫描的当前格子`cur`，每次循环将后面“格子”的物品数量加到前面的格子中，每次合并分三种情况，即合并后小于、等于或大于格子的最大容量，分别处理就行了。最后将打包的信息发给服务器，如果合并成功，服务器会返回一条消息`pack_Sortout`，我们接受后调用定义好的callback函数开始排序逻辑，而如果不需要进行任何合并，则直接调用该排序方法。

排序由`reqArrangePackData`方法完成，用`table.sort(t, func)`完成排序，其中`func`中定义了物品应根据何种方案决定先后顺序。注意这里还差一步，就是根据背包界面道具栏的格子宽度（横向有多个格子）计算出每个物品在道具栏中的位置坐标，完成这一步之后便可以给服务器发新的物品位置消息了。

### 一键删除

一键删除按钮绑定的事件要先弹出一个确认窗口，询问玩家是否丢弃，如果点击是则删除蓝色或蓝色以下的装备，如果点否则不做任何事。

这里就要用到类`UIPopBox`，它继承自`UIPanel`，由`UIManager`中的方法`getOrCreatePanel`创建，具体请看上面**任务5**的总结。

### 装备道具显示

道具装备的信息包括在背包中的格子坐标都是服务器给的，客户端只需要按照这个坐标去显示就行了。添加道具在`PackData`中的`_refreshItemData`方法中，可以分别用thisid或背包格子坐标的hash来索引（这个hash通过计算将x左移16位与y的和来得到）。

### 点选背包道具

背包种类根据枚举类型分为主背包、装备栏、时装等，主背包内的道具信息根据其格子坐标索引，装备栏道具信息则根据slotType索引，因为每个栏位是同一种类型的道具。点击道具后弹出的小窗是`UIBaseTips`的实例。

## 技能界面

`UIMainOperation`中定义各个技能按键，从配置文件读取技能id，为每个技能创建`UISkillBtn`对象，用技能id为其初始化，并添加到储存技能按键的节点上，按钮在创建时会根据传入的技能id注册相应的点击事件，回调函数中调用`AttackManager`中的`touchAttackBtn`方法，它会直接调用`AttackManager`中的`tryAttack`方法尝试攻击。玩家的操作是一个一个的事件，我们将这次释放技能或攻击的事件推入处理事件的队列`InputCommandQueue`，推入的对象按照事件的类型创建，比如我们这次应该创建一个`AttackEvent`，创建事件对象由专门的脚本`InputEventPool`完成。处理事件时，每帧`SceneManager`会在`update`方法中调用主角的`processInputEvent`方法，这个方法会一个个处理队列中的事件，每个事件依次进行，如果上一个事件还在执行，就不会处理下一个事件，事件的执行在`AttackEvent`自身的`onStart`方法，其实还是通过`AttackManager`来向服务器发送攻击消息（这里还有攻击过程上锁和解锁的过程，用来保证每次攻击完整完成再处理下一个攻击）。

消息发送后客户端的前半部分逻辑结束，后半部分就是服务器处理后发给客户端攻击结果的消息，客户端收到的消息包括：通知攻击动作、攻击结果、血量变化、npc死亡。

## 小弹窗优化

要求多个弹窗出现时后面出现的弹窗不覆盖前面的，而是点击之后依次出现前面的弹窗。那么实际上我们只需要一个窗口实例，将一个或多个弹窗信息储存在容器（例如栈）中，用户点击后就将窗口刷新成后面一个弹窗信息，直至容器为空，期间如果有新的弹窗，就压入栈，并刷新窗口至最新的弹窗信息。

处理弹窗点击结束（关闭弹窗）的逻辑要在回调函数之后，否则如果回调函数中更改了回调函数，就会覆盖这一次本该调用的回调。

栈中的消息处理，具体之后再整理
