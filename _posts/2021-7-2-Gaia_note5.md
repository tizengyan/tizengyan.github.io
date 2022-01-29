---
layout: post
title: "Gaia笔记5——CJ版本更新"
date: 2021-7-2
categories: UE4
tags: gaia
excerpt: 做的各种功能
author: Tizeng
published: true
---

* content
{:toc}

## 区分大厅和副本关卡（2021.7.2）

目前根据GameMode来区分，登录的关卡的StandAlone，可以拿到GameMode。

## Player初始化流程（2021.7.5）

碰撞、重力都在Avatar上设置，部分是引擎自带的，碰撞通道可以配置。

玩家身上带的武器和技能在PlayerState中初始化，收到服务器消息回调后，先从GameMode拿到地图id，读地图表拿到默认携带武器id数组，根据该数组初始化人物身上的武器Actor，具体的逻辑是由Character上的HoldingItemComponent去做，Spawn武器蓝图之后用AttachToComponent挂到人物的Mesh上。技能的初始化也是类似。

技能和武器关联，每次切换武器后触发的回调中进行了上技能的操作，激活技能后会进入相应的GA文件中执行`ActivateAbility`，并给玩家上特定的标签，UI中就是靠注册标签移除和添加的事件来判断是否激活了技能。

## 主动技能UI优化（2021.7.6）

将之前蓝图的逻辑移植到了lua脚本，之前用“AsyncTask...”中的各种“ListenFor...”方法注册人物身上某个Tag的增加、移除和StackChange，蓝图中直接在后面拖线写逻辑就行，代码要从接口返回的实例中显示的注册该Task中定义的Delegate，和普通Delegate一样，用`Add`即可。主要遇到两个问题：

* 监听某个Tag的StackChange的时候，第一次返回的并不是当前的Count而是1，因为激活技能后是先走通知逻辑，再走的给属性，这样激活后第一次的数据就不对了。解决方法是初始化时覆盖第一次的同步，使用该技能对应的初始属性值（如最大值）。
* 每个人物属性有专门的方法拿，不管是通过Player还是ASC都是。为了能够在lua中方便的通过字符串拿属性，我写了一个执行字符串的方法当作宏使用。

## 搬运物（2021.7.16）

要弹提示，发现缺少一个通用的给所有人提示的接口，这部分考虑移到GameState做。

## 队伍提示优化（2021.7.22）

由于要读表，代码里需要弹提示的地方全都是写死id的特殊处理，这块必须要改。

(2022.1.7)由于本地化的问题现在必须要重构这块逻辑了，因为部分读表是在ds上读好了传给客户端，这样会导致无法进行多语言的切换。

## 新养成界面（2021.8.2）

之前把设置界面抽象的基类中有TopBar的逻辑，现在养成界面也需要TopBar了，最好进一步将TopBar的逻辑抽象出来，如果有界面有这种功能就使用这个模块。

### 屏幕点击区域检测

ViewportScale：根据屏幕DPI换算的一个值，在编辑器中编辑umg时的坐标要乘以它才是最终的屏幕坐标，换言之触摸屏幕得到的屏幕坐标需要除以它来获得Canvas坐标
ViewportPosition：即屏幕坐标

## 红点系统（2021.8.13）

每个界面都可能有红点，同一个界面可能有多个地方有红点且相互独立（还可能重合），同一类型的红点还可能有多个，红点只有当满足一定条件时链路才会消失，否则一直存在。里面很多重复的逻辑，有必要抽象出来由一个Manager专门管理。

界面需要知道的信息：
* 要注册哪些通知来显示红点(初始化时)
* 现在有哪些红点立刻就要显示（初始化时）
* 什么时候刷新红点（如解锁新英雄）
* 什么时候清除红点（如点击）

至于收到通知之后创建红点、管理各个红点的显示与否，这些都交给Manager去管理。现在的Manager支持根据不同类型的红点，进行增删查改的操作，界面初始化的时候注册相应的事件刷新红点的状态，创建红点的位置由放好的NameSlot决定，因此注册的时候要告知Manager需要在哪个NameSlot下去创建，清除的时候也是去Clear掉这个Slot下的红点。当然很可能多个功能需要在同一处显示红点，一个功能的红点消失不代表该处的红点就要消失，这时还需要检查该NameSlot下还有没有其他需要显示红点的信息，在Manager中为每个NamgSlot进行计数即可，如果计数到0就说明需要清除。

## 副本任务（2021.9.1）

实际上是用现有的触发器框架，搭建一套任务系统。

任务提示UI通过专门显示UI的Task触发，我们为任务专门定义一个处理任务链的数据类型，将每个子任务所需的信息如任务名、描述、追踪Task等封装进一个结构TaskListData中，然后用一个TArray暴露给使用者配置。而当任务面板需要更多信息如击杀计数、倒计时、Marker等信息时，可以在相应功能的Task（如击杀计数）中定义delegate进行通知，相当于为每个需要计数的功能做特殊处理，但这样模块间会很耦合，不利于后期维护和扩展。最好是在TaskListData中再抽象一个ExternalData数据类，用于处理外部的各种信息，然后相应的功能继承这个基类去做实现。（这部分已经完成，ExternalData中定义了Initialize、Terminate、Reset等流程接口，具体的功能如计数、计时、进度等去继承它实现自己的功能逻辑，然后把数据通过Replicated成员同步到客户端，客户端再去根据通知的类型，拿到持有ExternalData的TaskListData中的UI实例，去刷新）

目前各个任务的UI是和每个Task相关的，如果同时触发多个会导致多个UI叠在一起。（这个已经通过在触发器中增加Widget是否唯一的配置解决）

### 计数

Editor编辑时有时会触发逻辑，可能在代码中残留数据，需要在OPROPERYTY中加上Transient标记，让引擎主动回收它。

计数目前直接放在TaskListData中，每个TaskData会去更新计数然后同步到客户端，显示ui，这样做的问题是无法支持多个Task显示，已经如果后面的配置和之前的Task一样，可能导致Replicated不通知的问题。（不光是计数，现在的任务链显示是不支持显示追踪多个任务的，只能一个一个来，这个要和关卡讨论一下具体的需求，到底是当前任务有若干个子任务，这些子任务完成后跳下一个任务，还是说要追踪多个不相干的任务去显示）

## HUD创建顺序

lua端实现了Controller的ReceiveAcknowledgePossession方法，服务器上当Controller被Possess之后，会调用客户端上的AcknowledgePossession，然后lua这边接到调用后创建HUD。

在拿SizeToContent的WidgetSize时，可以使用GetDesiredSize方法，但是可能返回0，原因是此Widget刚被添加数据还没拿到，可以在Tick里面去检测。

## 大厅角色动画适配（2021.10.15）

要为每个英雄配置不同的展示动画，包括idle、rest和升级时的levelup，idle在一段时间后会切换到rest，然后再回到idle，这些新逻辑需要新建一个动画蓝图（ABP），在蓝图中未动画的切换配置好条件（状态机），展示的切换较为简单，直接用一个Enum就行，然后创建一个AnimNotify蓝图，里面通过拿到ABP来设置Enum，配到相关动画资源中，这样使用了我们ABP的Mesh在动画播完的时候就会触发下一个动画了。

## 本地化（2021.10.19）

UE本身有一套Localization的逻辑，通过扫描资源和文本文件搜集所有Text信息，然后翻译成对应的语言存下来，在游戏中切换Culture的时候实时的切换文字。固定在ui上的Text比较好做，将需要显示的Text集中在一个StringTable中，搜集的时候就只用管一个目录下的资源而不用把所有资源全都查一遍。翻译可以使用UE自带的TranslationEditor，或者将内容导出为`.po`文件交给第三方，翻译完成后再导入。

这个流程遇到比较大的问题是lua中的文字信息，当初参照zt做多语言的做法，将所有中文信息集中到一个lua文件中，供其他脚本调用，但由于lua处理FText的时候将其转换成了string，就算在C++端定义了使用`NSLOCTEXT`宏的接口，lua这边得到的也是string而非FText，这样会导致在切换Culture的时候由lua设置的Text不会随着切换而变更语言。阅读源码得知，在切换Culture时，TextLocalizationManager会广播一个OnCulureChange的事件，而Text本身并不通过这个事件来刷新，它是在引擎的Tick中检测TextHistory的Revision是否过期，如果是则调用Rebuild去刷新。`GetDisplayString`接口被调用时也会根据Namespace和Key去找到当前的FTextId，通过它拿到Entry，然后检测该Entry中储存的信息是否和要拿取的信息一致（通过比较Hash值是否被改变），如果发现不一致，则调用改变Revision的方法。

### lua脚本适配

首先在GatherTextFromSourceCommandlet里面定义一个新的宏用来收集lua的文本，然后将unlua插件中用于处理TextProperty的逻辑修改一下，让其在对Text赋值前先调用`FText::FindText`，如果返回有值则说明可以进行本地化，赋值Find到的Text。注意这里用的key就是字符串本身，前面定义收集的时候会指定NameSpace和key，为了方便我们直接将文本自身作为key，这样在脚本中可以不用每个加宏的地方都去再定义一个key。

## 人物模型实时更换（2021.11.4）

新大厅支持玩家操控角色四处移动，在切换人物后，玩家控制的人物模型也要实时进行切换。这就需要将以前的Pawn销毁掉再重新创建，因为每个人物蓝图身上的配置不一样，如果只是更换Mesh会有隐患。

现在大厅能拉队友进来，换角色这个操作就会在ds上进行，然后同步到客户端，Possess之后摄像机会自动切到角色上，这个延迟会让选角界面的摄像机错位，思来想去应该在选角界面返回的时候再去换Avatar，而不是每次点击的时候。同时基地内Avatar初始化的时候有先获取id并接收更新的逻辑，如果一开始就拿到了正确的id，后面再收到的时候就不需要再更新了，总之就是保证只要在界面关闭时才去换Avatar，这样就不会有摄像机错位的问题。

## 军械库与局内商店（2021.12）

两个普通的需要页签分类点击各种小格道具，然后刷新信息的UI，细节就不写了，都大同小异，只记录遇到的问题。

商店由于是在战斗中用交互物打开，且每个交互物（商店）刷出的物品不同，这就需要场景中的Actor保存玩家刷出的道具数据，这块服务器写了个component，其中的商店数据用一个Replicated成员变量同步下来，如果ds上有改动会触发OnRep通知客户端刷新

涉及到颜色变化的地方，应该把颜色信息放在蓝图的变量里面，而不应该写在代码配置里，因为目前只有程序能修改脚本。

### 模型更换野指针

枪械模型展示和人物展示类似，都是先放了一个Actor在场景中，切换的时候根据Tag去找到它，然后在相同的位置Spawn对应的枪械并存下来，切换的时候就Spawn新的并把上一个Destroy，问题出现在这个操作由一个GameInstanceSubsystem管理的一个类来完成，切换场景后它并不会重新初始化，导致切换后还保留了之前创建武器Actor的指针（野），调用Destroy的时候就会crash。解决方法很简单，判指针的时候加一个使用`IsValidLowLevelFast`的判断就行了，如果内存异常它不会判过。

## 主动技能（2021.12.17）

使用下面的代码监听TagAdded时，出现过一段时间就监听不到了的情况：

```lua
local TagAddedRemovedIns = UAsyncTaskGameplayTagAddedRemoved.ListenForGameplayTagAddedOrRemoved(ASC, TagContainer)
if TagAddedRemovedIns then 
    TagAddedRemovedIns.OnTagAdded:Add(self, self.OnTagAdded)
end
```

这里用了local变量储存的AsyncTask会在一段时间后被gc，连带delegate当然就不会被通知了。
改成存`self`变量后，发现问依然存在，AsyncTask依然被gc了，尝试在WBP中先声明变量，然后lua直接存进去。最后发现是没有判断InputId，被其他技能的通知给盖掉了。

## 上漂提示系统优化（2022.1.9）

在做本地化的时候顺便优化了显示通知的接口，由于通知要区分玩家，之前的接口是直接放在Controller中，哪个玩家需要通知就在ds上调那个Controller的方法，把需要的字符串传入即可，这样的缺点是生成字符串的逻辑是在ds上进行的，导致多语言失效，另一个问题是生成消息的类之前处理消息时逻辑没有统一，每种消息都有一个方法去特殊处理，这样肯定是不行的，优化后支持多个参数进行Format，内容可以是读表的id也可以是直接传入的字符串（封进了一个结构），使用的时候按顺序放进数组即可。

## Bug合集

* 出现挂机之后大厅人物摄像头转移角度，之后操作会crash的问题，看日志是Controller中的PlayerCharacter为空，但此时并未掉线，通过在Controller的UnPossess处打断点发现Player由于掉出地图边界被销毁掉了
* 之前做的交互UI在Remove合AddItem时的表现出现异常，发现检查角度RemoveItem时高概率出现回正后grid不再出现的情况，排查很久之后发现是因为ListView自己缓存了widget的实例，在RemoveItem之后再次AddItem它可能直接使用了之前的widget而不去重新generate，所以OnEntryGenerated不会走，而widget被我删除之前会播一段消失动画，播完后widget就保留了不可见的状态被重新添加进ListView，造成明明Add了但就是看不见的情况。解决方法是每次添加完后手动拿到widget并检查可见性，如果不可见则播放出现的动画
* OnActorBeginOverlap用AddDynamic了之后不触发，发现是NPC基类没有勾选GenerateOverlapEvents选项，要触发Overlap事件必须两个接触的Actor都勾选了这个选项
* 正确注册Npc死亡事件后却一直没有触发，排查后发现每次第一次运行引擎会警告绑定失败，原因是不是UFUNCTION，但函数明明就定义为UFUNCTION，而且经排查只有这一个函数会出问题，后面再定义一个一模一样的函数绑定就会正常触发，最后注意到该函数声明前面有一个宏，将那个宏移开，之前的函数也能正常触发了，这里只能猜测是自己定义的宏和UFUNCTION宏之间出了点冲突
* C++定义的delegate广播的ustruct到了lua那边突然拿不到里面的成员，而且只在手机包出现，编辑器没有问题。调试发现是delegate传过来的struct类型莫名其妙变了，应该是unlua插件的问题，已经提交issue给他们，然后换了种实现回避掉这种情况
* 技能图标在某张地图上会出现收不到初始化通知，反复实验发现在手机上频繁出现，编辑器平均5次才出现1次，原因是ui初始化有时会慢于技能初始化通知，只要加一个主动获取的逻辑即可，注意由于ui只能拿到客户端数据，也就是要等它Replicated下来，而这个流程仍然可能晚于ui初始化，因此一定要在该数据的OnRep函数处去通知，而不是服务器初始化逻辑的地方
* UObject类自己实现的Replicated方法，在客户端经常无法收到绑定的OnRep函数调用。后面终于发现是ReplicateSubobjects实现写的有问题，没有查子object的更新导致
* 结算ui莫名被gc然后再被创建，然后Construct调用lua函数崩溃。发现是之前在ui脚本Destruct中增加的Destroy导致，由触发器创建的ui都会有这个问题，不能加Destroy。