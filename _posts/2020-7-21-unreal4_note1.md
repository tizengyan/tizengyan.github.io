---
layout: post
title:  "unreal4笔记1——上手"
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

## UMG（unreal motion graphics UI designer）

unlua相当于一个使用lua对umg进行操作的插件，优点是效率高、代码符合程序员直觉，逻辑复杂时不用担心太乱。缺点是不能打断点调试。

## 反射机制

很多语言如Java、lua都支持反射，

## Unlua

unlua调用引擎

引擎调用unlua，在引擎中创建蓝图后，生成unlua模板，编写脚本实现ui逻辑，蓝图中还要实现一个unlua接口，并注册蓝图的路径。

## 蒙太奇


