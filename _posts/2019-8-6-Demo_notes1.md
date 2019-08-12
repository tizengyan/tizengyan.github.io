---
layout: post
title:  "Demo笔记1——平台跳跃"
#date:   2019-03-01 8:10:54
categories: 游戏开发
tags: unity
excerpt: 开始做demo
author: Tizeng
---

* content
{:toc}

## Tilemap

用这个工具可以极大提高关卡制作效率，在不同的tilemap上可以设置不同的像素图，也可以为障碍物的tilemap加上Tilemap Collider2D来自动生成collider。

## 横版游戏角色的坑

横版游戏的角色与俯视视角的略有不同，首先只需要左右两个方向的移动，和往上方跳跃的操作，因此只需要接收“Horizontal”方向的变化。

在实现跳跃时不能简单的用`Addforce`，要先将RigidBody的y方向速度置为0，否则会与重力抵消一部分。

俯视视角时，移动方向是四周，因此用一个`moveDirection`的二维向量就可以表示出当前移动的方向，然后乘上速度，而横版游戏中这就会出现bug，因为我们不能再y轴增加任何速度，否则就相当于改变了重力的影响。下面是两个其他常见的bug：

* 跳跃后碰到其他地面会弹起

* 斜坡往下滑时一直播放Falling动画

这两个都是因为在写跳跃逻辑时出错或不准确导致。

我们把跳跃分为三个阶段：grounded、jumping、falling，在地面上时如果有跳跃输入，则进入jumping状态，此时角色在空中，且具有向上的速度，由重力影响不断减速，速度减为0后开始下落，进入falling状态，直到再次接触地面。不同的状态分别播放不同的动画，如果发现角色在某个情况（比如斜坡）一直播放下落动画，就说明其一直处于下落状态，这可能是角色脚下的判别器没有检测到地面，如下图所示：

![ground check](https://github.com/tizengyan/images/raw/master/ground_check.png)

为了防止这种情况，我们把ground check加到两个，一边一个同时检测，就不会出现这种情况了。

## 行为逻辑

描述：场景中有NPC，NPC会有一些行为，主角靠近之后可以对其附身，表现为主角的输入开始作用于该NPC，镜头也随之跟随，在特定情况（如该NPC死亡，或角色主动）下玩家回到操控主角的状态，镜头也随之切换。

### 处理输入

要实现这个功能，就要将主角和NPC的行为逻辑和输入逻辑分开，输入脚本只接收输入，不做任何操作，然后将输入传给行为脚本，行为脚本根据传过来的输入开始不同的行为，这样我们在需要附身时，只需要将输入脚本的输出从主角身上转移到对应的NPC身上，就可以实现操作的转移，而NPC没有被附身的时候，也可以用脚本生成简单的输入作为NPC自己的行为（AI），下面看具体代码：

```cs
using UnityEngine;

public class PlayerInput : MonoBehaviour {
    Controller controller;
    Vector2 moveDirection;
    bool isPressingSpace = false;

    void Start() {
        controller = GetComponent<Controller>();
    }

    void Update() {
        if(controller == null){
            return;
        }
        processInput();
        passInput();
    }

    public Controller getController() {
        return controller;
    }

    void processInput() {
        moveDirection.x = Input.GetAxisRaw("Horizontal"); // move
        //moveDirection.y = Input.GetAxisRaw("Vertical");
        if (Input.GetButtonDown("Jump")) { // jump
            isPressingSpace = true;
        }
    }

    void passInput() {
        controller.input(moveDirection.x, moveDirection.y, isPressingSpace);
        isPressingSpace = false;
    }
}
```

### 角色行为

为了复用部分代码，为角色的行为脚本定义一个基类Controller，后面不同的角色和NPC都继承这个基类，Input脚本在获取的时候只需要`GetComponent<Controller>`就行了：

```c#
using UnityEngine;

public class Controller : MonoBehaviour {
    [Range(1f, 20f)]
    public float speed = 10f;

    [HideInInspector]
    public Vector2 moveDirection;

    protected Rigidbody2D rb;
    protected bool facingRight = true;
    protected bool isPressingSpace = false;

    public void input(float x, float y, bool _isPressingSpace) {
        moveDirection.x = x;
        moveDirection.y = y;
        isPressingSpace = _isPressingSpace;
    }

    public void resetInput() {
        moveDirection = Vector2.zero;
        isPressingSpace = false;
    }
}
```

然后是主角的行为脚本，左右移动，落地时可以跳，不能二段跳：

```csharp
using System;
using UnityEngine;

public class PlayerController : Controller {
    [Range(1f, 30f)]
    public float jumpForce = 10f;

    public Animator animator;
    public Transform groundLeftCheck;
    public Transform groundRightCheck;
    public LayerMask ground;

    //private Rigidbody2D rb;
    //private Vector2 moveDirection;
    private float fallingSpeed = -7f;
    //private bool facingRight = true;
    private bool isGrounded = true;
    //private bool canDoubleJump = true;
    private bool isJumping = false;
    private bool isFalling = false;
    //private bool isJumping = false;

    private void Start() {
        rb = GetComponent<Rigidbody2D>();
    }

    private void FixedUpdate() {
        isGrounded = Physics2D.Linecast(transform.position, groundLeftCheck.position, ground) ||
                    Physics2D.Linecast(transform.position, groundRightCheck.position, ground);
    }

    private void Update() {
        //processInput();
        move();
        animate();
        //Debug.Log(moveDirection);
    }

    void processInput() {
        moveDirection.x = Input.GetAxisRaw("Horizontal"); // move
        moveDirection.y = rb.velocity.y; // gravity
        if (Input.GetButtonDown("Jump")) { // jump
            isJumping = true; // 进入跳跃状态
        }
    }

    public void move() {
        if (isPressingSpace) {
            isJumping = true;
        }
        moveDirection.x *= speed;
        moveDirection.y = rb.velocity.y; // gravity

        if ((rb.velocity.y < 1f && rb.velocity.y > -1f) && !isGrounded) { // 上升到最高点，纵向速度为0，跳跃状态结束（速度变化过快时失效
            isJumping = false;
        }
        if (moveDirection.y < -1f && !isGrounded) { // 速度为负，且不着地，进入下落状态
            isFalling = true;
        }

        if (isJumping && isGrounded) { // 在地上，起跳
            moveDirection.y = jumpForce;
        }
        //else if (canDoubleJump) {
        //    moveDirection.y = 15f;
        //    canDoubleJump = false;
        //}
        else if (moveDirection.y > 0.1f) { // 已起跳，加重力
            moveDirection.y = rb.velocity.y;
        }
        else if (isFalling) { // 开始下落，固定下落速度
            moveDirection.y = fallingSpeed;
        }
        rb.velocity = moveDirection; // 最后给刚体速度
    }

    public void animate() {
        animator.SetFloat("Horizontal", moveDirection.x);
        animator.SetBool("isFalling", isFalling);
        animator.SetBool("isJumping", isJumping);
        animator.SetBool("isGrounded", isGrounded);
        flip(moveDirection.x);
    }

    void flip(float horizontal) {
        if ((horizontal > 0 && facingRight == false) || (horizontal < 0 && facingRight == true)) {
            facingRight = !facingRight;
            transform.Rotate(0, 180f, 0);
        }
    }

    private void OnCollisionEnter2D(Collision2D collision) {
        if(collision.gameObject.tag == "Ground") {
            //canDoubleJump = true;
            isGrounded = true;
            isFalling = false;
            isJumping = false;
            animator.SetBool("isFalling", isFalling);
            animator.SetBool("isJumping", isJumping);
            animator.SetBool("isGrounded", isGrounded);
        }
    }
}
```

最后是一个NPC小鸟的行为脚本，小鸟的逻辑类似flappy bird，每按一下往上飞一点，没有输入就会下落，逻辑上其实就是可以无限跳跃，然后控制跳跃的力度：

```c#
using UnityEngine;

public class BirdController : Controller {
    [Range(1f, 20f)]
    public float jumpForce = 5f;

    Animator animator;

    void Start() {
        rb = GetComponent<Rigidbody2D>();
        animator = GetComponent<Animator>();
        moveDirection = Vector2.zero;
    }

    void Update() {
        move();
        animate();
    }

    void move() {
        moveDirection.x *= speed;
        moveDirection.y = rb.velocity.y;
        if (isPressingSpace) {
            moveDirection.y = jumpForce;
        }
        rb.velocity = moveDirection;
    }

    void animate() {
        flip(moveDirection.x);
    }

    void flip(float horizontal) {
        if ((horizontal > 0 && facingRight == false) || (horizontal < 0 && facingRight == true)) {
            facingRight = !facingRight;
            transform.Rotate(0, 180f, 0);
        }
    }
}
```

在主角没有附身小鸟的时候，小鸟需要有一些行为，这里简单起见，就让它不停的原地上下，为了实现这个效果需要调整每次自动跳跃的间隔，太大太小都不行，这里根据我的调整间隔应为0.76s：

```c#
using System.Collections;
using System.Collections.Generic;
using UnityEngine;

public class AI : MonoBehaviour {
    public float inputFrequency = 0.76f;

    Controller controller;
    bool isPressingSpace = false;

    void Start() {
        controller = GetComponent<Controller>();
        StartCoroutine(birdMove());
    }

    private void Update() {
        controller.input(0, 0, isPressingSpace);
        isPressingSpace = false;
    }

    IEnumerator birdMove() {
        while (true) {
            isPressingSpace = true;
            yield return new WaitForSeconds(inputFrequency);
        }
    }
}
```

### 附身技能

为了较低模块的耦合性，我将角色的附身技能这块独立出来，作为一个单独的脚本控制：

```c#
using System.Collections;
using System.Collections.Generic;
using UnityEngine;

public class PlayerSkill : MonoBehaviour {
    public new Camera camera;

    bool skillHeck;
    GameObject target;

    private void Start() {
        StartCoroutine(resetHeck());
    }

    private void Update() {
        //Debug.Log("is hecking = " + skillHeck);
    }

    IEnumerator resetHeck() {
        while (true) {
            skillHeck = false;
            yield return new WaitForSeconds(.1f);
        }
    }

    public void pressHeck() {
        skillHeck = true;
    }

    public void releaseHeck() {
        skillHeck = false;
    }

    public void heck() {
        if (GetComponent<PlayerInput>() == null || target.GetComponent<PlayerInput>() == null) {
            Debug.Log("PlayerInput does not exist");
            return;
        }
        GetComponent<PlayerInput>().getController().input(0, 0, false); // 重置输入，不然出错
        GetComponent<PlayerInput>().enabled = false;
        if(target.GetComponent<AI>() != null) {
            target.GetComponent<AI>().enabled = false;
        }
        target.GetComponent<PlayerInput>().enabled = true;

        // 对准摄像头
        if (camera == null) {
            Debug.LogError("No cam");
            return;
        }
        camera.GetComponent<CameraFollower>().setTarget(target);
    }

    public void retriveControl() {
        if(target == null) {
            Debug.LogError("No target");
            return;
        }
        target.GetComponent<PlayerInput>().enabled = false; // 外部：取消用户输入
        if (target.GetComponent<AI>() != null) { // 开启AI输入
            target.GetComponent<AI>().enabled = true;
        }
        GetComponent<PlayerInput>().enabled = true; // 主角：恢复用户输入
        target.GetComponent<Controller>().resetInput(); // 重置目标的输入
        target = null;
        camera.GetComponent<CameraFollower>().setTarget(gameObject);
    }

    private void OnTriggerStay2D(Collider2D collision) {
        if (collision.gameObject.tag == "Bird" && skillHeck) {
            target = collision.gameObject;
            heck();
            Debug.Log("Hecking");
        }
    }
}
```

至于摄像头对准这块，现在是直接把位置设过去，略显突兀，后面还需要优化。
