---
layout: post
title:  "Unity使用笔记3"
#date:   2019-03-01 8:10:54
categories: Unity3D
tags: unity
excerpt: 记录一下一些简单的脚本及其应用
author: Tizeng
---

* content
{:toc}

以下均以2D场景为例。

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

具体看注释

```c#
using System.Collections;
using System.Collections.Generic;
using UnityEngine;

public class BulletMover : MonoBehaviour {
    public Rigidbody2D rb;
    public float speed = 20f;
    public int damage = 50; // 设置子弹伤害

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
        // instantiate death effect
        GameObject clone = Instantiate(deathEffect, transform.position, transform.rotation);
        Destroy(clone, 0.5f); // 死亡特效自毁
        Destroy(gameObject);  // 销毁敌人对象
    }
}
```