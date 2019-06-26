---
layout: post
title:  "Unity使用笔记3——主角逻辑"
#date:   2019-03-01 8:10:54
categories: 游戏开发
tags: unity
excerpt: 记录一下一些简单的脚本及其应用
author: Tizeng
---

* content
{:toc}

以下均以2D场景为例。

## 主角移动

前面虽然实现了主角移动的脚本，但是没有配置相应的动画效果，也没有一个可以使之停下的方法，下面我们需要将移动的脚本进行完善。

```c#
using UnityEngine;

public class PlayerMover : MonoBehaviour {
    public float moveSpeed;
    public float SPEED = 20.0f;
    public Joystick joystick;
    public Animator animator;

    private Rigidbody2D rb;
    private Vector2 moveDirection;
    private bool isEnded = false;

    private void Start() {
        rb = GetComponent<Rigidbody2D>();
    }

    private void Update() {
        if (!isEnded) {
            ProcessInput();
            Move();
            Animate();
        }
        Debug.Log(moveDirection);
    }

    void ProcessInput() {
        //float moveHorizontal = Input.GetAxis("Horizontal");
        //float moveVertical = Input.GetAxis("Vertical");
        float moveHorizontal = joystick.Horizontal;
        float moveVertical = joystick.Vertical;
        moveDirection = new Vector2(moveHorizontal, moveVertical);
        moveSpeed = Mathf.Clamp(moveDirection.magnitude, 0.0f, 1.0f);
        moveDirection.Normalize();
    }

    void Move() {
        rb.velocity = moveDirection * moveSpeed * SPEED;
    }

    public void Stop() {
        moveDirection = new Vector2();
        rb.velocity = new Vector2();
        Animate();
        isEnded = true;
    }

    void Animate() {
        animator.SetFloat("Horizontal", moveDirection.x);
        animator.SetFloat("Vertical", moveDirection.y);
    }
}
```

## 子弹生成（prefabs）

具体看注释

```c#
using System.Collections;
using System.Collections.Generic;
using UnityEngine;

public class BulletSpawner : MonoBehaviour {

    public Transform firePoint;
    public GameObject bullet;

    void Update() {
        if (Input.GetButtonDown("Fire1")) // 检测输入
            spawnBullet();
    }

    void spawnBullet() {
        GameObject clone = Instantiate(bullet, firePoint.position, firePoint.rotation); // 生成子弹对象
        Destroy(clone, 2f);
    }
}
```

## 子弹移动

子弹不光需要在射击后往射击的方向移动，还需要检测自身的collider是否和敌人（或其他）的collider相撞，这通过定义`OnTriggerEnter2D`函数实现，如此一来我们可以得到与之碰撞的物体的属性，如果碰到的是敌人，那么就调用`Enemy`中的伤害函数，并自毁。

```c#
using System.Collections;
using System.Collections.Generic;
using UnityEngine;

public class BulletMover : MonoBehaviour {
    public Rigidbody2D rb;
    public float speed = 20f; // 子弹速度
    public int damage = 50;   // 子弹伤害

    void Start() {
        rb.velocity = transform.right * speed; // 初始化时设置速度
    }

    private void OnTriggerEnter2D(Collider2D hitInfo) {
        Enemy enemy = hitInfo.GetComponent<Enemy>(); // 检测是否与敌人collider相撞
        if (enemy != null) {
            enemy.takeDamage(damage); // 计算伤害
            Destroy(gameObject); // 子弹对象自毁
        }
    }
}
```

## 静态目标（敌人）

具体看注释

```c#
using System.Collections;
using System.Collections.Generic;
using UnityEngine;

public class Enemy : MonoBehaviour {
    public int health = 100; // 设置生命值
    public GameObject deathEffect; // 设置死亡特效

    public void takeDamage(int damage) {
        health -= damage;
        if (health <= 0) // 判断是否死亡
            die();
    }

    void die() {
        GameObject clone = Instantiate(deathEffect, transform.position, transform.rotation); // 生成死亡特效
        Destroy(clone, 0.5f); // 死亡特效自毁，延迟0.5s
        Destroy(gameObject);  // 销毁敌人对象
    }
}
```

## 锁定最近的敌人

游戏中主角和武器是两个不同的素材，我认为为了实现自动瞄准应该让武器去找敌人，然后转动自己指向它。这个过程分两步，一是找到最近的敌人，二是将我们的枪对准当前最近的敌人，我先实现了对准的功能，然后实现了找最近，为了将其合二为一，我将后者的代码整合进了前者功能的脚本，但是后来发现脚本间是完全可以互相调用内部的数据的，因此为了将实现细节隔离开增加程序的可读性和维护性，还是分了两个脚本。

至于如何调用另一个脚本中的数据，首先在当前脚本中建立目标脚本的临时对象，然后用`GetComponet<script name>()`抓取游戏对象上的脚本对其赋值，然后就可以用类的语法调用其中的公有成员。

### (1)找最近的敌人

要预先将所有敌人的`Tag`设置为`Enemy`。

```c#
public class FindCloest : MonoBehaviour {
    public float lookSpeed = 20f;
    private GameObject cloestEnemy = null;

    void Update() => findCloest();

    void findCloest() {
        cloestEnemy = null;
        float disMin = Mathf.Infinity;
        GameObject[] enemies = GameObject.FindGameObjectsWithTag("Enemy");
        foreach(GameObject cur in enemies) {
            float dis = (cur.transform.position - transform.position).magnitude;
            if (disMin > dis) {
                disMin = dis;
                cloestEnemy = cur;
            }
        }
        if(cloestEnemy != null)
            Debug.DrawLine(transform.position, cloestEnemy.transform.position);
    }

    public GameObject getCloest() {
        return cloestEnemy;
    }
}

```

## (2)瞄准

```c#
using UnityEngine;

public class Aim : MonoBehaviour {

    public float lookSpeed = 20f;

    FindCloest findCloest;
    Quaternion initialRotation;
    [SerializeField]
    GameObject target;
    bool isEnded = false;

    void Start() {
        findCloest = GetComponent<FindCloest>();
        initialRotation = transform.rotation;
    }

    void Update() {
        target = findCloest.getCloest();
        if (target != null) {
            Vector2 direction = target.transform.position - transform.position;
            float angle = Mathf.Atan2(direction.y, direction.x) * Mathf.Rad2Deg - 90;
            Quaternion rotation = Quaternion.AngleAxis(angle, Vector3.forward);
            transform.rotation = Quaternion.Slerp(transform.rotation, rotation, Time.deltaTime * lookSpeed);
        }
        else if(!isEnded){ // 如果场上没有敌人，摆正武器
            transform.rotation = initialRotation;
            isEnded = true;
        }
    }
}
```

## 血条

首先在主角脚本`PlayerMover`中加入`takeDamage`和`die`两个函数，但要注意最后不能摧毁主角，否则会有bug，然后把血条素材用一个Image加入场景（注意不是Sprite），然后将其属性设置为Filled，就可以在扣血的脚本中随心所欲的从各个方向调整血条的百分比了。

```c#
public Image healthBar;

...
healthBar.fillAmount = health / maxHealth;
...
```

注意在赋值的时候一定要保证后面的数字是`float`类型。

## Debug日志

### (1)分数重启后会累积

造成这个的原因是记录分数的变量`totalScore`是静态的，声明周期是整个程序运行周期，因此重新载入关卡并不会影响它的累积，我们需要在显示记录分数的`ScoreCounter`脚本开头把它设为0，这样每次载入脚本时分数会从0开始计算。至于为什么要把`totalScore`定义成静态的，因为我的计分方式是每消灭一个敌人分数就+10，而敌人是否被消灭要依靠敌人身上的脚本`Enemy`来判断，这就需要在敌人对象的脚本中累积分数，将`totalScore`定义成静态，就可以直接用`ScoreCounter.totalScore`来完成这个操作，不然的话，就需要在`Enemy`脚本中创建一个`ScoreCounter`的临时对象，然后再用`FindGameObjectWithTag`来获取附有计分脚本的`UI`对象，再用`GetComponent`获取上面的计分脚本对象，最后才能通过临时对象对其中的成员`totalScore`进行操作。

### (2)人物在游戏结束后依然移动

当消灭所有敌人后，当前关卡完成，隐藏游戏UI，显示结算界面UI，但如果消灭最后一个敌人的时候屏幕上的摇杆不在原点（这几乎必定会发生，因为都会边走边打），那么控制角色移动的脚本`PlayerMover`中的`joystick`的横纵方向就会储存下最后一瞬间玩家推动摇杆的方向，从而在每帧调用`Move`函数时造成角色依然往那个方向移动。

要解决，首先需要在游戏结束后停止`Update`内的任何调用，这可以用一个简单的布尔变量实现，同时，由于移动是用设置物体速度的方式实现的，因此也要把速度置为0，但这样做之后角色虽然会在游戏结束后停止移动，但依然会播放移动的帧动画，所以我们要把动画状态也重置一次。
