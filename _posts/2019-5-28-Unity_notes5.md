---
layout: post
title:  "Unity使用笔记5——NPC寻路"
#date:   2019-03-01 8:10:54
categories: 游戏开发
tags: unity
excerpt: A*寻路算法在Unity中的实现
author: Tizeng
---

* content
{:toc}

本文记录A*寻路算法在Unity中的实现，教程来自YouTube博主Sebastian Lague，[地址在此](https://www.youtube.com/watch?v=nhiFx28e7JY&list=PLFt_AvWsXl0cq5Umv3pMC9SPnKjfp9eGW&index=2)，无字幕。

## 构建地图

在实现寻路算法之前，我们先要规定好地图的定义方式，首先定义一个`Node`类，它决定一个节点是否为障碍物，和在世界中的绝对坐标。另外还有A*算法中需要用到的路径消耗，这里先忽略。

```c#
using UnityEngine;

public class Node {
    public bool walkable;
    public Vector2 worldPosition;
    public int gCost, hCost;

    public int fCost {
        get { return gCost + hCost; }
    }

    public Node(bool _walkable, Vector2 _worldPosition) {
        walkable = _walkable;
        worldPosition = _worldPosition;
    }

    public bool equals(Node node) {
        return (this.walkable == node.walkable && this.worldPosition == node.worldPosition) ? true : false;
    }
}
```

定义好节点后，下一步是定义网格，我们用前面定义的`Node`二维数组来储存网格，也就是地图，同时我们还需要以下信息：

* 地图的大小，这里用一个二维向量储存

* 节点半径，虽说是半径，但每个节点其实是用方形来表示，这里只是方便说明

* 障碍物的 layer 信息，用`LayerMask`变量储存

* 最后调试时，需要主角的位置信息

* 最后是节点的直径，也就是长度，以及以这个长度为单位的size

下面来看代码，首先在`Start`中初始化直径以及`gridSizeX`和`gridSizeY`，这两个的值是地图总长与节点长度的比值取整，最后调用创建网格的函数。

创建网格时，以之前计算出的`girdSizeX`和`gridSizeY`为基准，初始化二维数组`grid`，然后计算出地图左下角顶点的世界坐标，用以循环时计算每个节点的中心坐标。创建每个节点时直接调用构造函数，而其中的`walkable`成员根据当前位置的半径内是否存在Collider且该物体是否带有障碍物的 layer 而决定。剩下的就是简单的数学问题了。

接着为了把处于地图中的人物位置，转化成地图网格`grid`中的节点信息，需要一个函数来实现，大致的思路是给定一个位置的坐标，我们假定它在地图内且地图为一个矩形，先计算该坐标对比于地图长宽的比例，以x轴为例，如果越接近地图左边那么x轴的比例就越接近0%，越靠右就越接近100%，同样的，月靠上相对于y轴的比例就越接近100%。回想一下，我们的地图是用一个节点的二维数组`grid`储存的，因此我们需要的是以节点长为单位的在`grid`中的横纵坐标，就可以得到当前世界坐标下，我们的物体处于哪个节点上，想清楚这点后，无非就是将刚刚计算出的x、y轴百分比乘以`gridSizeX`和`gridSizeY`后取整，然后带入`grid`就行了。

最后的一个函数`OnDrawGizmos`既不属于游戏逻辑也不属于寻路逻辑，而是为了将上面的分析和计算结果可视化，便于我们debug。

下面的代码中会有一些在这个步骤不需要用到的变量和方法，暂时忽略它们。

```c#
using UnityEngine;

public class Grid : MonoBehaviour {
    Node[,] grid; // 网格数组
    public Vector2 gridWorldSize; // 总网格大小
    public float nodeRadius = 0.5f; // 每个节点所占范围
    public LayerMask unwalkableMask;
    public Transform player;

    float nodeDiameter;
    int gridSizeX, gridSizeY; // 以大网格中的小方格为单位的size

    void Start() {
        nodeDiameter = nodeRadius * 2;
        gridSizeX = Mathf.RoundToInt(gridWorldSize.x / nodeDiameter);
        gridSizeY = Mathf.RoundToInt(gridWorldSize.y / nodeDiameter);
        CreateGrid();
    }

    void CreateGrid() {
        grid = new Node[gridSizeX, gridSizeY];
        Vector2 worldLeftBottom = transform.position - Vector3.right * gridSizeX / 2 - Vector3.up * gridSizeY / 2; // 通过向量坐标计算得到大网格内左下角的绝对坐标

        // 在每个网格内检查是否存在collider
        for(int i = 0; i < gridSizeX; i++) {
            for(int j = 0; j < gridSizeY; j++) {
                Vector2 worldPoint = 
                    worldLeftBottom + 
                    Vector2.right * (i * nodeDiameter + nodeRadius) + 
                    Vector2.up * (j * nodeDiameter + nodeRadius);
                // 检查的范围半径就是我们前面定义的 gridRadius

                //bool walkable = !(Physics.CheckSphere(worldPoint, nodeRadius, unwalkableMask));
                bool walkable = !(Physics2D.OverlapCircle(worldPoint, nodeRadius, unwalkableMask));
                grid[i, j] = new Node(walkable, worldPoint);
            }
        }
    }

    public Node nodeFromWorldPoint(Vector2 worldPosition) {
        float percentX = (worldPosition.x + gridSizeX / 2) / gridWorldSize.x; // 这里不能用 gridSizeX，因为它是以nodeDiameter为单位的
        float percentY = (worldPosition.y + gridSizeY / 2) / gridWorldSize.y;
        percentX = Mathf.Clamp01(percentX);
        percentY = Mathf.Clamp01(percentY);

        int x = Mathf.RoundToInt(percentX * gridSizeX);
        int y = Mathf.RoundToInt(percentY * gridSizeY);
        return grid[x, y];
    }

    private void OnDrawGizmos() {
        Gizmos.DrawWireCube(transform.position, new Vector3(gridWorldSize.x, gridWorldSize.y, 1));

        if (grid != null) {
            Node playerNode = nodeFromWorldPoint(player.position); // 得到主角坐标转换成的节点
            foreach (Node node in grid) {
                Gizmos.color = (node.walkable == true) ? Color.white : Color.red; // 障碍物染成红色，其他染成白色
                if (node.equals(playerNode))
                    Gizmos.color = Color.blue; // 如果找到了该节点，把该节点染成蓝色
                Gizmos.DrawWireCube(node.worldPosition, Vector3.one * (nodeDiameter - .1f)); // 画出立方体，Vector3.one代表三个方向大小都为1的单位向量
            }
        }
    }
}
```

可以看到，角色坐标位置的方格被染成蓝色，并会随着角色移动而移动：

![grid_player_position](https://github.com/tizengyan/images/raw/master/grid_player_position.png)

## A*算法实现

准备工作算是做完了，下面可以实现寻路算法了。寻路算法最常见的有广度优先、深度优先、dijkstra等，在游戏中用的比较多的是A*，还有对其的各种优化方法比如IDA*，我们后面会提，这里先实现A*算法的核心思路。

### (1)寻路

算法思路为：

从起点开始，遍历相邻的所有节点，将其

计算节点间的距离

回溯路径

![Astar_getDistance](https://github.com/tizengyan/images/raw/master/Astar_getDistance.png)

```c#

```

下面是实现效果：

![Astar_pathfinding1](https://github.com/tizengyan/images/raw/master/Astar_pathfinding1.png)

![Astar_pathfinding2](https://github.com/tizengyan/images/raw/master/Astar_pathfinding2.png)

### (2)NPC沿路追逐

## Debug日志

### (1)网格障碍物无法显示

构建地图时没有理解检测collider实现的逻辑，而很久没有将障碍物用红色网格显示在scene view，原因首先是教程视频中的场景为3D，障碍物的collider也是3D的，而我的场景是2D的且是2D Collider，因此用`Physics.CheckSphere`是检测不出的，而要用`Physics2D.OverlapCircle`，才能顺利识别出2D的Collider。
识别出了Collider还没完，还需要确认这个被识别出的物体是不能够穿行的障碍物，这就需要用到 layer，我是第一次用`LayerMask`在代码中标记物体，所以一直忘记在脚本界面中为`LayerMask`变量选取正确的 layer，也难怪会识别不出障碍物了。debug后效果图如下所示：

![grid_obstacles1](https://github.com/tizengyan/images/raw/master/grid_obstacles1.png)

用红色方块标记的障碍物：

![grid_obstacles2](https://github.com/tizengyan/images/raw/master/grid_obstacles2.png)

### (2)寻路时出现绕路

整体没问题，能找到路，就是不是最短路径，第一个想到的问题就是计算节点间距离的函数出了问题，一看果然，赋值的时候有个X忘了改成Y，导致部分距离计算错误。

### (3)网格绘制时出现偏移

地图网格会根据设置的网格大小往左下角或右上角移动，而不是切合在地图的平面，这个明显是地图左下角的世界坐标计算出错，原因在于这里本应该用原始的系统坐标的长宽来计算，因为要得到的是坐标，我在计算时误用了以网格大小为单位的长宽值。

### (4)物体坐标转化为网格坐标时出现偏移

这个问题出在转换函数`nodeFromWorldPoint`上，计算坐标所在位置在系统中所占地图长宽百分比时，再次误用了以网格大小为单位的长宽值而造成。有趣的是，如果把网格半径设置为0.5，上述错误就不会发生，因为此时网格边长正好为1，与系统坐标单位长度相吻合，事实上，只要网格边长为1的整数倍，都不会出现这个bug。这也说明在跟着教程走时还不够细心，才会把相似的变量名称写错。

### (5)用heap优化寻路算法时Unity死机

判断应该是进入了死循环，首先查看定义的堆中的两个while(true)，