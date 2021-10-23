---
layout: post
title: "Gaia笔记5——CJ版本更新"
date: 2021-7-2
categories: UE4
tags: gaia
excerpt: 做的各种功能
author: Tizeng
published: false
---

* content
{:toc}

## 区分大厅和副本关卡（2021.7.2）

目前根据GameMode来区分

## Player初始化流程（2021.7.5）TODO

位置、碰撞、重力

玩家身上带的武器和技能在PlayerState中初始化，收到服务器消息回调后，先从GameMode拿到地图id，读地图表拿到默认携带武器id数组，根据该数组初始化人物身上的武器Actor，具体的逻辑是由Character上的HoldingItemComponent去做，Spawn武器蓝图之后用AttachToComponent挂到人物的Mesh上。技能的初始化也是类似。

技能和武器关联，每次切换武器后触发的回调中进行了上技能的操作，激活技能后会进入相应的GA文件中执行`ActivateAbility`，并给玩家上特定的标签，UI中就是靠注册标签移除和添加的事件来判断是否激活了技能。

## 主动技能UI优化（2021.7.6）

将之前蓝图的逻辑移植到了lua脚本，之前用“AsyncTask...”中的各种“ListenFor...”方法注册人物身上某个Tag的增加、移除和StackChange，蓝图中直接在后面拖线写逻辑就行，代码要从接口返回的实例中显示的注册该Task中定义的Delegate，和普通Delegate一样，用`Add`即可。主要遇到两个问题：

* 监听某个Tag的StackChange的时候，第一次返回的并不是当前的Count而是1，因为激活技能后是先走通知逻辑，再走的给属性，这样激活后第一次的数据就不对了。解决方法是初始化时覆盖第一次的同步，使用该技能对应的初始属性值（如最大值）。
* 每个人物属性有专门的方法拿，不管是通过Player还是ASC都是。为了能够在lua中方便的通过字符串拿属性，我写了一个执行字符串的方法当作宏使用。

## 搬运物（2021.7.16）TODO

要弹提示，发现缺少一个通用的给所有人提示的接口
提示不应该和Controller相关，这部分考虑移到GameState做##

## 队伍提示重构（2021.7.22）TODO

由于要读表，代码里需要弹提示的地方全都是写死id的特殊处理

## 新养成界面（2021.8.2）

之前把设置界面抽象的基类中有TopBar的逻辑，现在养成界面也需要TopBar了，最好进一步将TopBar的逻辑抽象出来，如果有界面有这种功能就使用这个模块。

### 屏幕点击区域检测

FGeometry、ViewportScale、IsUnderLocation
AbsolutePosition 
LocalPosition 
ViewportPosition 

## 红点系统（2021.8.13）

每个界面都可能有红点，同一个界面可能有多个地方有红点且相互独立（还可能重合），同一类型的红点还可能有多个，红点只有当满足一定条件时链路才会消失，否则一直存在。里面很多重复的逻辑，有必要抽象出来由一个Manager专门管理。

界面需要知道的信息：
* 要注册哪些通知来显示红点(初始化时)
* 现在有哪些红点立刻就要显示（初始化时）
* 各个红点清除的条件（如点击）

至于收到通知之后创建红点

## 副本任务（2021.9.1）

实际上是用现有的触发器框架，搭建一套任务系统

任务提示UI通过专门显示UI的Task触发，我们为任务专门定义一个处理任务链的数据类型，将每个子任务所需的信息如任务名、描述、追踪Task等封装进一个结构TaskListData中，然后用一个TArray暴露给使用者配置。而当任务面板需要更多信息如击杀计数、倒计时、Marker等信息时，可以在相应功能的Task（如击杀计数）中定义delegate进行通知，相当于为每个需要计数的功能做特殊处理，但这样模块间会很耦合，不利于后期维护和扩展。最好是在TaskListData中再抽象一个ExternalData数据类，用于处理外部的各种信息，然后相应的功能继承这个基类去做实现。

目前各个任务的UI是和每个Task相关的，如果同时触发多个会导致多个UI叠在一起。

### 计数

Editor编辑时有时会触发逻辑，可能在代码中残留数据，需要在OPROPERYTY中加上Transient标记，让引擎主动回收它。

计数目前直接放在TaskListData中，每个TaskData会去更新计数然后同步到客户端，显示ui，这样做的问题是无法支持多个Task显示，已经如果后面的配置和之前的Task一样，可能导致Replicated不通知的问题。

## HUD创建顺序

lua端实现了Controller的ReceiveAcknowledgePossession方法，服务器上当Controller被Possess之后，会调用客户端上的AcknowledgePossession，然后lua这边接到调用后创建HUD。

在拿SizeToContent的WidgetSize时，可以使用GetDesiredSize方法，但是可能返回0，原因是此Widget刚被添加数据还没拿到，可以在Tick里面去检测。

## 队友行为上漂整理（2021.9.27）

## 大厅角色动画适配（2021.10.15）
要为每个英雄配置不同的展示动画，包括idle、rest和升级时的levelup，idle在一段时间后会切换到rest，然后再回到idle，这些新逻辑需要新建一个动画蓝图（ABP），

## 本地化（2021.10.19）

UE本身有一套Localization的逻辑，通过扫描资源和文本文件搜集所有Text信息，然后翻译成对应的语言存下来，在游戏中切换Culture的时候实时的切换文字。固定在ui上的Text比较好做，将需要显示的Text集中在一个StringTable中，搜集的时候就只用管一个目录下的资源而不用把所有资源全都查一遍。翻译可以使用UE自带的TranslationEditor，或者将内容导出为`.po`文件交给第三方，翻译完成后再导入。

这个流程遇到比较大的问题是lua中的文字信息，当初参照zt做多语言的做法，将所有中文信息集中到一个lua文件中，供其他脚本调用，但由于lua处理FText的时候将其转换成了string，就算在C++端定义了使用`NSLOCTEXT`宏的接口，lua这边得到的也是string而非FText，这样会导致在切换Culture的时候由lua设置的Text不会随着切换而变更语言。阅读源码得知，在切换Culture时，TextLocalizationManager会广播一个OnCulureChange的事件，而Text本身并不通过这个事件来刷新，它是在引擎的Tick中检测TextHistory的Revision是否过期，如果是则调用Rebuild去刷新。`GetDisplayString`接口被调用时也会根据Namespace和Key去找到当前的FTextId，通过它拿到Entry，然后检测该Entry中储存的信息是否和要拿取的信息一致（通过比较Hash值是否被改变），如果发现不一致，则调用改变Revision的方法。

## Bug合集

* 出现挂机之后大厅人物摄像头转移角度，之后操作会crash的问题，看日志是Controller中的PlayerCharacter为空，但此时并未掉线，通过在Controller的UnPossess处打断点发现Player由于掉出地图边界被销毁掉了
* 之前做的交互UI在Remove合AddItem时的表现出现异常，发现检查角度RemoveItem时高概率出现回正后grid不再出现的情况，排查很久之后发现是因为ListView自己缓存了widget的实例，在RemoveItem之后再次AddItem它可能直接使用了之前的widget而不去重新generate，所以OnEntryGenerated不会走，而widget被我删除之前会播一段消失动画，播完后widget就保留了不可见的状态被重新添加进ListView，造成明明Add了但就是看不见的情况。解决方法是每次添加完后手动拿到widget并检查可见性，如果不可见则播放出现的动画
* OnActorBeginOverlap用AddDynamic了之后不触发，发现是NPC基类没有勾选GenerateOverlapEvents选项，要触发Overlap事件必须两个接触的Actor都勾选了这个选项
* 正确注册Npc死亡事件后却一直没有触发，排查后发现每次第一次运行引擎会警告绑定失败，原因是不是UFUNCTION，但函数明明就定义为UFUNCTION，而且经排查只有这一个函数会出问题，后面再定义一个一模一样的函数绑定就会正常触发，最后注意到该函数声明前面有一个宏，将那个宏移开，之前的函数也能正常触发了，这里只能猜测是自己定义的宏和UFUNCTION宏之间出了点冲突
* C++定义的delegate广播的ustruct到了lua那边突然拿不到里面的成员，而且只在手机包出现，编辑器没有问题
* 