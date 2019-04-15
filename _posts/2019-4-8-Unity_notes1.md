---
layout: post
title:  "Unity基础整理（待完成）"
#date:   2019-03-06 12:12:54
categories: 基础知识
tags: unity
excerpt: 整理一些Unity3D的用法
author: Tizeng
---

* content
{:toc}

## Scene

我理解为游戏的关卡。

## GameObject

游戏内与玩家互动的物体或场景。官方文档的说明为："Base class for all entities in Unity Scenes."

系统中每个物件都有自己的坐标系统，如果某个物件是另一个物件的从属（孩子），那么它的坐标系原点就始终是另一个物件的相对于世界坐标系的坐标，好比地球绕太阳转，而月球绕地球转。

后面出现GameObject我会一律说物体。

## Component

官方文档中做如下描述：

Base class for everything attached to GameObjects.

Note that your code will never directly create a Component. Instead, you write script code, and attach the script to a GameObject.

即它是构成`GameObject`的各种元素。

## material

决定游戏中物体的样子，可以理解为包裹在物体外的表皮。同一个材质可以影响多个物体，但是如果修改这个材质会反应在左右应用了这个材质的物体上。它本质上也是一个 component。

### skybox

这是一种特殊的材质，专门用于创建游戏中的背景，即所处世界的材质。有多种类型，以六面体类型举例，想象我们置身于一个六面体中，然后可以根据自己的喜好往不同的墙面上贴上材质贴图。创建完毕后需要在unity中选择 Window->Lighting->settings 加入。

## camera

摄像机在引擎中分两类，一种是perspective（透视的），和真实世界中的相机无异，都是把三维的物体投影到二维的平面，还有一种是orthographic（直线的），在这种相机拍到的场景中不存在透视效果，也就是说无论离摄像机多远，看起来都一样大，这种相机并不真实存在，但是可以为我们创建某些效果提供便利。

还有一点区别，透视摄像机取景是从上下左右四个面由里向外辐射的形状（形状像一个从上往下看的金字塔），这个金字塔也有底，叫做 clipping plane，任何在这个平面之后的物体都会被忽略不进入视野，我们可以手动调节这个平面的距离，换句话说，我们只能看到在这个金字塔内部的物体。

而直线摄像机则是一个直直的长方体，无限向外延伸。

同一个场景中可以有多个摄像机，我们可以通过控制它们在屏幕中显示的长宽比来调整看到的画面，也可以通过设置优先级来决定哪一个摄像机拍摄的画面要优先显示。我们可以利用它来建立小地图。

## MonoBehavior

MonoBehaviour是脚本的基类，一般来说用C#写脚本时第一件事就是继承MonoBehavior，这样就可以使用其中的方法如Start、Update。

而Component是Behavior和MonoBehavior的基类，它决定了脚本可以和GameObject链接在一起，Behavior继承自Component，只要继承Behavior的类都可以被设置为enabled或disabled，如果我们不希望一个类可以被这样设置，就不应该继承Behavior（比如Rigidbody和Collider，它们直接继承自Component），而MonoBehavior继承自Behavior，因此只要继承了MonoBehavior的类就可以设置为enabled或disabled，有时我们并不希望自己的类可以被这样设置，但又需要用到MonoBehavior提供的函数（比如Invoke），就会遇到两难的境地。

以一个让物体绕另一个物体旋转的脚本为例：

```c#
using UnityEngine;
using System.Collections;

public class RotateAround : MonoBehaviour {

    public Transform target; // the object to rotate around
    public int speed; // the speed of rotation

    void Start() {
        if (target == null) {
            target = this.gameObject.transform;
            Debug.Log ("RotateAround target not specified. Defaulting to parent GameObject");
        }
    }

    // Update is called once per frame
    void Update () {
        // RotateAround takes three arguments, first is the Vector to rotate around
        // second is a vector that axis to rotate around
        // third is the degrees to rotate, in this case the speed per second
        transform.RotateAround(target.transform.position, target.transform.up, speed * Time.deltaTime);
    }
}
```

`RotateAround`是unity提供的API，输入的参数有三个，分别为：绕之旋转的坐标（可以是自身，即自转）、旋转的坐标轴、速度。其中速度以度（degree）为单位。

`Time.deltaTime`得到的是当前帧和前一帧的时间间隔。例如我们如果需要让物体沿着y轴每秒钟移动n个单位，那么只需要每帧往y坐标加上 n × Time.deltaTime 即可。

### Invoke

用以在特定时间触发特定的函数。第一个参数接受要触发的函数名，第二个参数为等待的时间。类似的，`InvokeRepeating`可以不停的重复这一操作，比如我们需要在游戏中不断的生成敌人或物体，就可以有如下脚本：

```c#
using UnityEngine;
using System.Collections;

public class InvokeRepeating : MonoBehaviour {

    public GameObject target;

    void Start() {
        InvokeRepeating("SpawnObject", 2, 1);
    }

    void SpawnObject() {
        float x = Random.Range(-2.0f, 2.0f);
        float z = Random.Range(-2.0f, 2.0f);
        Instantiate(target, new Vector3(x, 2, z), Quaternion.identity);
    }
}
```

### coroutine