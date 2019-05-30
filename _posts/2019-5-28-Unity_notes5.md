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

```c#
using System.Collections.Generic;
using UnityEngine;

public class MyGrid : MonoBehaviour {
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

    public int maxSize {
        get { return gridSizeX * gridSizeY; }
    }

    void CreateGrid() {
        grid = new Node[gridSizeX, gridSizeY];
        Vector2 worldLeftBottom = transform.position - Vector3.right * gridWorldSize.x / 2 - Vector3.up * gridWorldSize.y / 2; // 通过向量坐标计算得到大网格内左下角的绝对坐标

        // 在每个网格内检查是否存在collider
        for(int i = 0; i < gridSizeX; i++) {
            for(int j = 0; j < gridSizeY; j++) {
                Vector2 worldPoint = 
                    worldLeftBottom + 
                    Vector2.right * (i * nodeDiameter + nodeRadius) + 
                    Vector2.up * (j * nodeDiameter + nodeRadius);
                // 检查的范围半径就是我们前面定义的 gridRadius

                //bool walkable = !(Physics.CheckSphere(worldPoint, nodeRadius, unwalkableMask));
                bool walkable = !(Physics2D.OverlapCircle(worldPoint, nodeRadius, unwalkableMask)); // 注意这里应该用Physics2D
                grid[i, j] = new Node(walkable, worldPoint, i, j);
            }
        }
    }

    public Node nodeFromWorldPoint(Vector3 worldPosition) {
        float percentX = (worldPosition.x + gridWorldSize.x / 2) / gridWorldSize.x; // 这里不能用 gridSizeX，因为它是以nodeDiameter为单位的
        float percentY = (worldPosition.y + gridWorldSize.y / 2) / gridWorldSize.y;
        percentX = Mathf.Clamp01(percentX);
        percentY = Mathf.Clamp01(percentY);

        int x = Mathf.RoundToInt(percentX * gridSizeX);
        int y = Mathf.RoundToInt(percentY * gridSizeY);
        return grid[x, y];
    }

    public List<Node> getNeighbours(Node node) {
        List<Node> neighbours = new List<Node>();
        for(int i = -1; i <= 1; i++) {
            for(int j = -1; j <= 1; j++) {
                if (i == 0 && j == 0)
                    continue; // 忽略自身
                int checkX = node.gridX + i;
                int checkY = node.gridY + j;
                if(checkX >= 0 && checkX < gridSizeX && checkY >= 0 && checkY < gridSizeY) 
                    neighbours.Add(grid[checkX, checkY]);
            }
        }
        return neighbours;
    }

    List<Node> path;
    public void setPath(List<Node> _path) => path = _path; // 寻路时用来储存路径

    private void OnDrawGizmos() {
        Gizmos.DrawWireCube(transform.position, new Vector3(gridWorldSize.x, gridWorldSize.y, 1));

        if (grid != null) {
            Node playerNode = nodeFromWorldPoint(player.position);
            foreach (Node node in grid) {
                Gizmos.color = (node.walkable == true) ? Color.white : Color.red;
                if (node == playerNode) // or equals?
                    Gizmos.color = Color.blue;
                else if (path != null && path.Contains(node)) {
                    Gizmos.color = Color.yellow;
                }
                Gizmos.DrawWireCube(node.worldPosition, Vector3.one * (nodeDiameter - .1f));
            }
        }
    }
}
```

可以看到，角色坐标位置的方格被染成蓝色，并会随着角色移动而移动：

![grid_player_position](https://github.com/tizengyan/images/raw/master/grid_player_position.png)

## A*算法实现

准备工作算是做完了，下面可以实现寻路算法了。寻路算法最常见的有广度优先、深度优先、dijkstra等，在游戏中用的比较多的是A*，还有对其的各种优化方法比如IDA*，我们后面会提，这里先实现A*算法的核心思路以及简单的优化。

### (1)寻路

首先我们需要一个标准对每个节点进行评估，A*算法中每个节点定义两个cost，分别是`gCost`和`hCost`，前者是该节点到起点的距离，后者是该节点到目标节点也就是终点的距离，这两者相加就是`fCost`总估值，我们希望这个值越小越好，越小就意味着从起点到终点的消耗越小。

下面是A*算法思路：

1. 建立openSet和closedSet两个容器，分别储存需要检查的节点和已经检查过了的节点，并把起始节点放进openSet；

2. 循环（当openSet不为空时）：

    (1) 取出openSet中`fCost`最小的节点，记为`currentNode`（如果相等，则取`hCost`最小的），并将它放入closedSet；

    (2) 检查`currentNode`是否为目标节点，如果是则路径找到，调用`retracePath`并结束循环，如果不是则继续；

    (3) 对于`currentNode`的每个相邻节点（最多为8个）记为`neighbor`依次进行以下操作：

    判断其是否为障碍物或已经存在于closedSet中，如果是则忽略，如果不是则计算`gCost`值，为`currentNode.gCost`加上`neighbor.gCost`，注意这里可能出现搜索到重复节点的情况，因此得到的`gCost`有可能比之前得到的小，这说明找到了一条比之前更短的路径，因此这里也一并更新了`neighbor`的`gCost`，同时，将它的`parent`更新为`currentNode`。最后如果节点`neighbor`是不存在于openSet中的新节点，还需要将其添加进openSet。

    (4) 返回(1)。

下面是A*算法代码：

```c#
void findPath(Vector3 startPos, Vector3 targetPos) {
    Stopwatch sw = new Stopwatch();
    sw.Start(); // 计时

    Node startNode = grid.nodeFromWorldPoint(startPos);
    Node targetNode = grid.nodeFromWorldPoint(targetPos);

    //List<Node> openSet = new List<Node>();
    MyHeap<Node> openSet = new MyHeap<Node>(grid.maxSize); // heap优化
    HashSet<Node> closedSet = new HashSet<Node>();

    openSet.Add(startNode); // 初始化openSet

    while (openSet.Count > 0) {
        Node currentNode = openSet.removeFirst();

        // 线性搜索
        //Node currentNode = openSet[0];
        //for (int i = 1; i < openSet.Count; i++)
        //    // 选择较小的FCost的neighbor，如果相等，则选择HCost小的，即离终点近的
        //    if (openSet[i].fCost < currentNode.fCost || (openSet[i].fCost == currentNode.fCost && openSet[i].hCost < currentNode.hCost))
        //        currentNode = openSet[i];
        //openSet.Remove(currentNode);

        closedSet.Add(currentNode);

        if (currentNode == targetNode) {
            sw.Stop();
            print("Path found: " + sw.ElapsedMilliseconds + "ms");

            retracePath(startNode, targetNode);
            return;
        }

        foreach(Node neighbor in grid.getNeighbours(currentNode)) {
            if(neighbor.walkable == false || closedSet.Contains(neighbor)) 
                continue;
            int new_gCost = currentNode.gCost + getDistance(currentNode, neighbor);
            if (new_gCost < neighbor.gCost || openSet.Contains(neighbor) == false) {
                neighbor.gCost = new_gCost;
                neighbor.hCost = getDistance(neighbor, targetNode);
                neighbor.parent = currentNode;
                if (openSet.Contains(neighbor) == false)
                    openSet.Add(neighbor);
            }
        }
    }
}
```

计算节点间的距离时，我们有横纵移动和对角移动两种移动方式，假设每个节点间的间隔为1，那么横纵间距就是1，斜对角距离就是根号2，约为1.414，这里为了方便近似成1.4，且都扩大十倍

![Astar_getDistance](https://github.com/tizengyan/images/raw/master/Astar_getDistance.png)

由上图可以看出，总距离是边长较短一边长度大小的斜线距离（图种为2）加上长边长与短边长的差值的直线距离（图种为5-2=3），由此我们可以写出计算节点距离的函数：

```c#
int getDistance(Node node1, Node node2) {
    int disX = Mathf.Abs(node1.gridX - node2.gridX);
    int disY = Mathf.Abs(node1.gridY - node2.gridY);
    if (disX > disY)
        return 14 * disY + 10 * (disX - disY);
    return 14 * disX + 10 * (disY - disX);
}
```

寻路完毕后最后一步是回溯路径，这也是为什么我们在节点中需要`parent`成员保存上一个节点，这样在寻路完成后，可以通过目标节点一步一步还原出整个路径，就像一个链表的头指针。

```c#
void retracePath(Node startNode, Node endNode) {
    List<Node> path = new List<Node>();
    Node curNode = endNode;
    while(curNode != startNode) {
        path.Add(curNode);
        curNode = curNode.parent;
    }
    path.Reverse();
    grid.setPath(path);
}
```

下面是实现效果：

![Astar_pathfinding1](https://github.com/tizengyan/images/raw/master/Astar_pathfinding1.png)

![Astar_pathfinding2](https://github.com/tizengyan/images/raw/master/Astar_pathfinding2.png)

### (2)排序优化

上面在寻路时每次都要检查每个节点的`fCost`，复杂度为O(n)，效率很低，而我们每次需要的是节点中`fCost`最小的那个，很自然想到用最小堆来优化，之前写堆排序已经很熟了，思路就不赘述，只是这次用C#实现一个堆而已：

```c#
using System;

public class MyHeap<T> where T : IHeapItem<T> { // 注意这里是T继承而不是MyHeap继承
    T[] items;
    int curSize;

    public MyHeap(int maxSize) {
        items = new T[maxSize];
    }

    public bool Contains(T item) {
        return Equals(items[item.index], item);
    }

    public int Count {
        get { return curSize; }
    }

    public void updateItem(T item) {
        siftUp(item);
    }

    public void Add(T item) {
        item.index = curSize;
        items[curSize] = item;
        siftUp(item);
        curSize++;
    }

    public T removeFirst() {
        T first = items[0];
        curSize--;
        items[0] = items[curSize];
        items[0].index = 0;
        siftDown(items[0]); // !!
        return first;
    }

    void siftUp(T item) {
        int parentIndex = (item.index - 1) / 2;
        while (true) {
            T parentItem = items[parentIndex];
            if (item.CompareTo(parentItem) > 0) { // 优先级更高
                swap(parentItem, item);
                parentIndex = (item.index - 1) / 2;
            }
            else
                break;
        }
    }

    void siftDown(T item) {
        int minIndex = item.index;
        while (true) {
            int rightIndex = item.index * 2 + 2;
            int leftIndex = item.index * 2 + 1;
            if (leftIndex < curSize) {
                if (rightIndex < curSize) {
                    if (items[minIndex].CompareTo(items[rightIndex]) < 0)
                        minIndex = rightIndex;
                }
                if (items[minIndex].CompareTo(items[leftIndex]) < 0)
                    minIndex = leftIndex;

                if (minIndex == item.index) {
                    break;
                }
                else {
                    swap(items[minIndex], item);
                }
            }
            else
                break;
        }
    }

    void swap(T item1, T item2) {
        items[item1.index] = item2;
        items[item2.index] = item1;
        // 最后不要忘了交换下标信息
        int temp = item1.index;
        item1.index = item2.index;
        item2.index = temp;
    }
}

public interface IHeapItem<T> : IComparable<T> {
    int index { get; set; }
}
```

要注意的是，这是一个泛型类，可以支持任何种类的数据，为了方便比较，定义一个储存下标的接口，它继承自`IComparable`，并让类型`T`实现，这样我们就可以方便的调用每个节点在堆中的下标。每次交换位置时也需要注意，不只要交换两个节点在数组中的位置，还需要交换它们的在数组中的**下标信息**。

然后将寻路代码中的openSet设置为我们定义的堆，每次用`removeFirst`进行提取最优节点并维护堆的性质，这样只需要O(logn)复杂度，在地图较大和单位较多的情况下可以显著提升效率。

同时不要忘了在`Node`类中加入下面的代码，实现我们定义的接口：

```c#
int heapIndex;
public int index {
    get {
        return heapIndex;
    }
    set {
        heapIndex = value;
    }
}

public int CompareTo(Node node) {
    int res = fCost.CompareTo(node.fCost);
    if (res == 0)
        res = hCost.CompareTo(node.hCost);
    return -res; // 我们需要小的消耗，因此越小优先级越高
}
```

下面看一下对比，以30x30大小的地图为例，节点半径设为0.05（边长为0.1），找到路径的时间消耗：

![Astar_without_heap.png](https://github.com/tizengyan/images/raw/master/Astar_without_heap.png)

可以看到用堆优化后速度快了十倍左右，效果非常明显：

![Astar_with_heap.png](https://github.com/tizengyan/images/raw/master/Astar_with_heap.png)

### (3)NPC沿路追逐

关于如何在有寻路算法的条件下，让NPC沿着计算出的路径移动并追逐，由于篇幅所限，放在了前一篇博客中。

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

判断应该是进入了死循环，首先查看定义的堆中的两个while(true)方法，`siftDown`和`siftUp`，设置断点检查，发现并无跳出问题，而在`siftDown`方法中进入一个反复对同一个节点进行操作的死循环，判断是调用的上层函数`removeFirst`出了问题，怀疑没有交换数组中第一个元素和最后一个元素，后来发现是把取出的第一个元素重新放入了`siftDown`函数，而不是传入已经换过来了的末尾元素，更改之后问题解决。就这一个小问题花了三个小时debug，说明写代码时随意变更变量不是个好习惯。