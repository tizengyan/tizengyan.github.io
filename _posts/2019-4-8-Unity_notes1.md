---
layout: post
title:  "Unity使用笔记1——内置函数"
#date:   2019-03-06 12:12:54
categories: 游戏开发
tags: unity
excerpt: 整理一下Unity3D中的内置方法
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

### FixedUpdate

在每个锁定帧（fixed frame-rate frame）调用，与物理系统的计算频率相同（默认为0.02s，即每秒钟调用50次），`Update`则完全取决于运行游戏时当前的帧数。当我们需要用`Rigidbody`时通常会用到。

### Awake

当脚本被载入的时候会调用`Awake`，被用来在游戏开始前初始化，整个脚本生命周期只会运行一次，但它在所有对象都被初始化后被调用，并且每个脚本中`Awake`的调用顺序是随机的，因此它可以被用来设置脚本之间的一些关联。

## Quaternion

这个类用来表示空间中的旋转，它替我们处理空间中复杂的有关旋转的运算，常用的函数有： Quaternion.LookRotation, Quaternion.Angle, Quaternion.Euler, Quaternion.Slerp，下面看一些示例

Quaternion.Euler(float x, float y, float z)返回一个绕x轴转x度，绕y轴转y度，绕z轴转z度的rotation

```c#
// A rotation 30 degrees around the y-axis
Quaternion rotation = Quaternion.Euler(0, 30, 0);
```

Quaternion.Slerp(Quaternion a, Quaternion b, float t)在a、b两个rotation中制造大小为t的改动，具体使用后面再说。

## 角度和弧度的转换

弧度转角度：

```c#
float angle = Mathf.Atan2(direction.y, direction.x) * Mathf.Rad2Deg
```

角度转弧度使用`Mathf.Deg2Rad`，它和`(PI * 2) / 360`等价。

## OnDrawGizmos

这是一个Unity定义的接口，如果我们需要为某一个物体（GameObject）周围绘制出某种图像信息方便调试，可以去自己实现这个接口，其中的方法多由`Gizmos`开头。比如第五章实现寻路算法时，地图的方格表现就用了这个接口。

## LayerMask

在Unity编辑器中最多可以使用32种 LayerMask，前八种是内置的 Layer，后面24种由用户自行定义，然后用`true`、`false`来决定是否识别该 Layer，通常被用来在 Raycast时判断是否遇到某一类 Layer 的物体。

## Canvas

创建任何UI列表下的对象都会自动生成Canvas并被收纳为其子项，且在子项中越靠下的对象越先被渲染。

## Coroutines

在Unity中，每一帧会处理完所有脚本中的代码，然后载入下一帧，但有时我们可能希望某些脚本中的函数不在同一帧内全部运行完，而希望有所延迟，比如一个脚本判断场上的敌人和玩家间的距离是否小于某个值，如果我们让程序每一帧都运行这个脚本且场上有众多搭载了这个脚本的单位的话，程序运行的效率会变得非常慢，很明显，我们用不着每一帧都去检测这个距离，而只需要将检测的频率提高到某个阈值，就不影响游玩体验，比如每0.1s检测一次，这时就需要用到 coroutines。

默认情况下，一个函数一旦被调用就会一直运行，直到运行完毕或在中途 return，coroutine允许我们在函数运行中途暂停，并在特定延迟之后从上次暂停的状态继续运行，有点像单片机中的中断。具体语法如下：

```c#
IEnumerator Fade() {
    for (float f = 1f; f >= 0; f -= 0.1f) {
        Color c = renderer.material.color;
        c.a = f;
        renderer.material.color = c;
        yield return null;
    }
}
```

这段代码每一帧减小一次物体的透明度。首先函数的返回类型为`IEnumerator`，然后在函数体某处有`yield return`关键词，如果直接接`null`，那么函数中断后，**下一帧**就会接着上次中断的状态运行下一行代码，如果需要中断自定义长的时间，那么只需把返回值从`null`改成`new WaitForSeconds(your delay)`即可。下面的代码运行后就会每隔3秒检查一下颜色的状态：

```c#
private SpriteRenderer sr;
public Color color1;
public Color color2;

void Start () {
   sr = GetComponent<SpriteRenderer>();
   StartCoroutine(ChangeColor());
}

IEnumerator ChangeColor() {
    while (true) {
        if (sr.color == color1)
            sr.color = color2;
        else
            sr.color = color1;
        yield return new WaitForSeconds(3);
    }
}
```

`StartCoroutine`和`StopCoroutine`分别用于启动和终止coroutine。

## Action

这是`System`中的一个用来使用无返回值委托的关键字。
