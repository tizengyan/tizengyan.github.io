---
layout: post
title:  "Unity使用笔记4——NPC逻辑"
#date:   2019-03-01 8:10:54
categories: 游戏开发
tags: unity
excerpt: NPC的行为实现
author: Tizeng
---

* content
{:toc}

## 0.NPC状态切换

NPC在游戏中的行为通常用状态机（state machine）实现，可以一定程度上和控制动画间关系的Animator联系起来。

## 1.NPC追逐

简单的无脑靠近实现很简单，利用Unity的状态机定义behavior脚本，然后将其与Animator中的相应动画状态绑定即可：

```c#
using UnityEngine;

public class ChaseBehavior : StateMachineBehaviour {
    public Transform playerPosition;
    public float chasingSpeed = 2f;

    // OnStateEnter is called when a transition starts and the state machine starts to evaluate this state
    override public void OnStateEnter(Animator animator, AnimatorStateInfo stateInfo, int layerIndex) {
        playerPosition = GameObject.FindGameObjectWithTag("Player").transform;
    }

    // OnStateUpdate is called on each Update frame between OnStateEnter and OnStateExit callbacks
    override public void OnStateUpdate(Animator animator, AnimatorStateInfo stateInfo, int layerIndex) {
        animator.transform.position = Vector2.MoveTowards(animator.transform.position, playerPosition.position, chasingSpeed * Time.deltaTime);
    }
}
```

但是这样体验并不好，NPC不是穿过障碍物就是被障碍物卡住，我们需要NPC可以合理的绕过障碍接近玩家。

## 2.避障算法

关于寻路算法详见下一篇，这里将直接调用`Astar`脚本。

## 3.NPC寻路

首先需要一个单独的脚本来规划每次的寻路请求，我们叫它`PathManager`，和`Astar`脚本共同附在一个场景物件上。

首先定义一个结构体`PathRequest`封装每个寻路请求，它储存起始坐标、目标坐标和 callback 函数，再用一个队列来处理可能出现的一系列寻路请求。静态函数`requestPath`让我们可以从外部之前发送寻路请求，它首先新建一个请求，将其加入队列，然后调用`tryProcessNext`尝试处理，如果同时满足：1.当前未在处理（用一个 bool 变量`isProcessing`记录）2.队列中至少存在一个请求，则弹出队列中的第一个请求，并调用`Astar`中的寻路方法开始寻路，最后不要忘了将`isProcessing`置为`true`以防止重复处理。

下面是具体代码：

```c#
using System.Collections.Generic;
using UnityEngine;
using System;

public class PathManager : MonoBehaviour {

    Queue<PathRequest> pathRequests = new Queue<PathRequest>();
    PathRequest currentPathRequest;

    Astar astar_pathFinding;
    bool isProcessing;

    static PathManager instance;

    void Awake() {
        instance = this;
        astar_pathFinding = GetComponent<Astar>();
    }

    public static void requestPath(Vector3 pathStart, Vector3 pathEnd, Action<Vector3[], bool> callback) {
        PathRequest newRequest = new PathRequest(pathStart, pathEnd, callback);
        instance.pathRequests.Enqueue(newRequest);
        instance.tryProcessNext();
    }

    void tryProcessNext() {
        if(isProcessing == false && pathRequests.Count > 0) {
            currentPathRequest = pathRequests.Dequeue();
            isProcessing = true;
            astar_pathFinding.findPathStart(currentPathRequest.pathStart, currentPathRequest.pathEnd);
        }
    }

    public void finishedProcessing(Vector3[] path, bool isSuccess) {
        currentPathRequest.callback(path, isSuccess);
        isProcessing = false;
        tryProcessNext();
    }

    struct PathRequest {
        public Vector3 pathStart;
        public Vector3 pathEnd;
        public Action<Vector3[], bool> callback;

        public PathRequest(Vector3 _pathStart, Vector3 _pathEnd, Action<Vector3[], bool> _callback) {
            pathStart = _pathStart;
            pathEnd = _pathEnd;
            callback = _callback;
        }
    }
}
```

同时，`Astar`中的`findPath`方法需要改为一个 coroutine，这样可以不让搜索带来的性能损耗导致帧数下降。找到路径后，再向`PathManager`反应结果，调用其中的`finishedProcessing`函数，它会调用当前寻路请求中的 callback 函数（它在`Unit`脚本中调用`requestPath`时会被指定，后面会提到），然后尝试处理下一个请求。

### 给每个需要寻路的单位分配任务

这个任务由`Unit`脚本完成，我们需要将其附在每个需要追逐玩家的NPC上。

在`Start`中初始化`StartCoroutine(updatePath())`后，单位的寻路逻辑就会开始执行，经过计算一系列初始变量如警戒范围、最小寻路阈值等后进入死循环，为了让其每隔一定时间开启寻路请求，`updatePath`函数被定义为 Coroutine，这样就不会在死循环中卡死，如果与目标距离在阈值范围内，则调用`requestPath`请求寻路，其中的 callback 函数为`onPathFound`，用以在路径找到后调用移动函数`followPath`，它同样也是 Coroutine，由于可能不停的在更新寻路请求重新寻路，因此在`StartCoroutine(followPath())`前先要`StopCoroutine(followPath())`以避免出错。

```c#
using System.Collections;
using UnityEngine;

public class Unit : MonoBehaviour {
    public Transform targetTransform;
    public float followSpeed = 1f;
    public bool showPath = true;
    public float alertRange = 8f;
    int targetIndex;
    Vector3[] path;
    Vector3 currentWayPoint;// debug
    const float minPathUpdateTime = 1f;
    const float moveThreshold = 0.5f;
    Animator animator;

    private void Awake() {
        targetTransform = GameObject.FindGameObjectWithTag("Player").transform;
    }

    void Start() {
        //PathManager.requestPath(transform.position, targetTransform.position, onPathFound);
        StartCoroutine(updatePath());
        animator = GetComponent<Animator>();
    }

    IEnumerator updatePath() {
        if (Time.timeSinceLevelLoad < .3f) {
            yield return new WaitForSeconds(.3f); // 防止刚刚启动场景时初始几帧的卡顿
        }

        float currentDis = (transform.position - targetTransform.position).sqrMagnitude; // check range
        if (currentDis < alertRange * alertRange) // 注意这里比较的是距离的平方
            PathManager.requestPath(transform.position, targetTransform.position, onPathFound);

        float sqrMoveThreshold = moveThreshold * moveThreshold;
        Vector3 targetPosOld = targetTransform.position;

        while (true) {
            yield return new WaitForSeconds(minPathUpdateTime);
            currentDis = (transform.position - targetTransform.position).sqrMagnitude; // check range
            if ((targetTransform.position - targetPosOld).sqrMagnitude > sqrMoveThreshold && currentDis < alertRange * alertRange) {
                PathManager.requestPath(transform.position, targetTransform.position, onPathFound);
                targetPosOld = targetTransform.position;
            }
        }
    }

    public void onPathFound(Vector3[] newPath, bool isSuccess) {
        if (isSuccess) {
            path = newPath;
            StopCoroutine(followPath());
            StartCoroutine(followPath());
        }
    }

    IEnumerator followPath() {
        if (path.Length > 0) {
            currentWayPoint = path[0];
            targetIndex = 0; // !!!
            while (true) {
                if (transform.position == currentWayPoint) {
                    targetIndex++;
                    if (targetIndex >= path.Length) {
                        //PathManager.requestPath(transform.position, targetTransform.position, onPathFound);
                        yield break;
                    }
                    currentWayPoint = path[targetIndex];
                }

                // 判断左移还是右移
                if(currentWayPoint.x > transform.position.x) {
                    animator.SetBool("isRight", true);
                }
                else {
                    animator.SetBool("isRight", false);
                }

                transform.position = Vector3.MoveTowards(transform.position, currentWayPoint, followSpeed * Time.deltaTime);
                yield return null;
            }
        }
    }

    public void OnDrawGizmos() {
        if(path != null && showPath) {
            for(int i = targetIndex; i < path.Length; i++) {
                Gizmos.color = Color.yellow;
                Gizmos.DrawCube(path[i], Vector3.one / 2);
                if (i == targetIndex)
                    Gizmos.DrawLine(transform.position, path[i]);
                else
                    Gizmos.DrawLine(path[i - 1], path[i]);
                // debug
                Gizmos.color = Color.cyan;
                Gizmos.DrawCube(currentWayPoint, Vector3.one);
            }
        }
    }
}
```

## Debug日志

### (1)部分路径中有障碍物边缘

首先想到简化路径函数中回溯时出了问题，纸上画图跑过算法后发现的确如此，`simplifyPath`函数中的`wayPoints.Add(path[i].worldPosition)`应该改成`wayPoints.Add(path[i - 1].worldPosition)`，作者应该是这里打错了，导致虽然得到的路径是正确的，但是存入的`wayPoints`往后差了一个网格单位，这虽然不是个很明显的bug，但是在障碍物为多边形而非矩形的时候会表现的尤为明显，寻路的NPC经常会因此卡在半路。

### (2)NPC只在开始的寻路

调试的时候发现NPC永远只走到玩家的初始位置，而之后不管玩家如何移动都不会再次寻路。本质上是因为在`Unit`脚本下的`findPath`方法中，`targetIndex`进入循环之前应该被重置为0，否则开启第二次寻路时，判断条件会一直跳出。

最骚的是，我一开始并没有发现是这个问题，而以为是把`requestPath`这个静态方法放在了`Start`里，所以只寻路一次，由于当时还没有修改`targetIndex`这个问题，因此就算把它放入`update`也无济于事，而实际上我们不需要这样做。

### (3)当主角或寻路目标为障碍物网格时，NPC开始原地抽搐

应该是在寻路算法中出现了无论如何都不能找到路径的死循环

### (4)NPC在某个节点停止寻路后，之后就不会再次开始寻路

发现这些点多为障碍物附近，推测是障碍物的辐射半径过大，导致周围网格也被标记为障碍物，一旦NPC在这些区域停留，程序会认为起点为障碍物，在`findPath`中就永远不会开始寻路。选择显示网格调试后果然如此。

### (5)NPC在变换路径过后似乎会加速

目前怀疑是由于使用了函数`MoveTowards`的问题，如果朝同一个方向多次调用这个函数，其速度会越来越快。目前暂时还没有解决。

### (6)代码复用效率不高

这虽然不是个bug，但是会影响工作效率，类似的功能以前已经写过代码，但是要用到不同场景中时发现难以复用，只能新建脚本复制粘贴，这样不是长久之计，一定要学会尽量实现功能的同时，让其Generic。
