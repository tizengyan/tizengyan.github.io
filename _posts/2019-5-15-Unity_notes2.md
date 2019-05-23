---
layout: post
title:  "Unity使用笔记2"
#date:   2019-03-01 8:10:54
categories: Unity3D
tags: unity
excerpt: 记录一下unity的使用技巧
author: Tizeng
---

* content
{:toc}

## 1.Tag & Layers

选中GameObject后状态栏的第二排有这两种属性可以设置，有一些预设的 tag ，也可以自己加，注意此处的 Layer 并不决定哪个层会被渲染在前面。下面还可以找到另一个属性 Sorting Layer，它才是控制2D游戏中各个物体的前后关系的，越往下层的会越先被渲染，挡住上层的物体。

## 2.Animation

Unity中的动画效果非常关键，首先选择需要创建动画的物体，点 Window->Animation 调出动画窗口，然后依次创建需要用到的动画，以备后面使用。记得在创建完毕后点 Inspector 中的 Apply，就会保存改动到模板中，如果后面需要调试，只需先选中目标物体，再在动画窗口选择相应的动画效果播放、调试即可。

## 3.Animator

类似状态机的设置，将上面制作的动画通过创建的变量设置转移的条件和结构。

* Any State状态：代表任意的状态，当我们希望某个条件发生时物体不论在何种状态都转移至另一种状态时可以用到，比如角色死亡

* Entry状态：初始化的状态

一般来讲会在需要构建动画的sprites上创建Animator，最好再额外创建一个Animator Controller并将其拖拽进Animator。在状态机编辑器中，若要使用Any State状态，在创建transition时务必注意取消勾选Can Transition To Self，否则会造成死循环，出现目标动画永远只播放第一帧的bug。

还有要注意的一点是，不同状态间的转换路径transition中的条件，是and关系，也就是如果有多个条件，必须同时满足时才会触发，而如果需要or条件，创建多个transition即可。

## 基础2D脚本

### (1)角色移动

2D场景中角色根据游戏的模式有两种移动方式，分别为俯视视角自由移动和平台跳跃。这里先介绍第一种，即俯视视角时角色可以朝至少四个方向移动的脚本。有一种较为简单的写法是捕捉键盘上的方向键如WASD，然后根据键位改动角色的位置值(`transform`)，虽然这种写法不需要给物体加上rigidbody2d，但是比较萎，我们偏向另一种更标准的写法。

```c#
using System.Collections;
using System.Collections.Generic;
using UnityEngine;

public class PlayerMover : MonoBehaviour {
    public float speed;
    private Rigidbody2D rb;

    private void Start() {
        rb = GetComponent<Rigidbody2D>(); // 拿到rigidbody2d
    }

    private void FixedUpdate() {
        float moveHorizontal = Input.GetAxis("Horizontal");
        float moveVertical = Input.GetAxis("Vertical");
        Vector2 movement = new Vector2(moveHorizontal, moveVertical);
        rb.AddForce(movement * speed); // 施加力
    }
}
```

这种写法需要在物体上加上rigidbody2d，然后根据输入向水平或垂直方向施加力，从而实现移动。注意2D环境下系统默认是横版跳跃视角，因此会有一个向下的重力，我们在实现俯视视角时需要把重力设置为0。

它的缺点是按键完后物体由于施加了力，会一直移动，而且如果按住不放会越来越快。
因此直接施加力体验并不好，我们可以改为直接设置速度，下面是最终的代码：

```c#
using System.Collections;
using System.Collections.Generic;
using UnityEngine;

public class PlayerMover : MonoBehaviour {
    public float speed;
    private Rigidbody2D rb;
    private Vector2 movement;

    void Start() {
        rb = GetComponent<Rigidbody2D>();
    }

    void FixedUpdate() {
        ProcessInput();
        Move();
    }

    void ProcessInput() {
        float moveHorizontal = Input.GetAxis("Horizontal");
        float moveVertical = Input.GetAxis("Vertical");
        movement = new Vector2(moveHorizontal, moveVertical);
    }

    void Move() {
        rb.velocity = movement * speed;
    }
}
```

### (2)摄像头跟随

这也一个很基础的脚本，在游戏中经常会用到，其实实现这个功能可以直接将摄像机和 player 设置为父子关系，但是这样会有一个缺点，一旦角色发生转动摄像机也会随之发生转动，造成视觉灾难，因此一般用脚本实现。

```c#
using System.Collections;
using System.Collections.Generic;
using UnityEngine;

public class CameraFollower : MonoBehaviour{

    public GameObject player;
    private Vector3 offset;

    // Start is called before the first frame update
    void Start(){
        offset = transform.position - player.transform.position;
    }

    void LateUpdate(){
        transform.position = player.transform.position + offset;
    }
}
```

`transform.position`代表这个脚本附着的对象的位置，在我们的例子中即是摄像机的位置。
`LateUpdate`与`Update`一样也会每帧被调用，不同的是它总发生在`Update`之后，也就是说我们在其中改变摄像机位置时，player 一定已经移动在先。核心代码其实只有一行，就是将摄像机的位置设置为当前 player 的位置，但是不能直接设置，因为尽管是2D场景，但是摄像机依然有三维的坐标，因此如果直接将摄像机摆在 player 位置，会导致镜头内拍不到任何东西。`offset`就是为此而存在，注意它只在`start`函数中计算，这时还没有发生任何移动，因此这个差值就是摄像机在同步到 player 位置后需要补上的Z轴坐标，我们需要将其加上。

### (3)Joystick Pack组件

首先创建一个公有的Joystick变量`joystick`，然后将之前移动脚本的`Input.GetAxis("Horizontal")`换成`joystick.Horizontal`即可。