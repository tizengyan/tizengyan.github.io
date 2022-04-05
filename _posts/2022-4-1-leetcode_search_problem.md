---
layout: post
title:  "算法题——搜索类问题"
date:   2022-04-1 16:11:54
categories: 算法题
tags: algorithm
excerpt: 需要用到DFS、BFS等搜索算法的问题
author: Tizeng
---

* content
{:toc}

## 全排列（[LeetCode 46](https://leetcode-cn.com/problems/permutations/)）

这题主要难点有三：
    1.想到用dfs来解决问题
    2.递归时使用for循环来构建搜索树
    3.每次递归后要进行撤回操作，保证同一层级在搜索时使用的数据是一样的，代码中表现的就是pop出答案的元素，或将标志位置回。

```c++
vector<vector<int>> res;
void dfs(vector<int>& nums, unordered_map<int, int>& visited, vector<int>& v)
{
    if (v.size() == nums.size())
    {
        res.push_back(v);
        return;
    }
    for (int i = 0; i < nums.size(); ++i)
    {
        if (visited[nums[i]] != 1)
        {
            visited[nums[i]] = 1;
            v.push_back(nums[i]);
            dfs(nums, visited, v);
            v.pop_back();
            visited[nums[i]] = 0;
        }
    }
}

vector<vector<int>> permute(vector<int>& nums)
{
    if (nums.size() == 0)
        return res;
    unordered_map<int, int> visited;
    vector<int> v;
    dfs(nums, visited, v);
    return res;
}
```

## 岛屿问题（[LeetCode 200](https://leetcode-cn.com/problems/number-of-islands/)）

给定一个二维数组，其中`0`代表海洋`1`代表陆地，只有横纵两个方向相邻才算接壤，一片相邻的陆地称为岛屿，计算岛屿的数量。

同样用dfs解决，但是要想到每次进行一次dfs后都要将已经走过的地方做好标记，避免重复搜索，然后根据需要在适当的地方添加逻辑，比如最基础的这道题，每次看到陆地就进行一次dfs，那么整片相邻陆地就会被标记，计数加1，接着重复操作。

岛屿问题有很多变种，如求最大岛面积（695）、岛屿周长（463）、最大人工岛（827）等，其中比较有趣的是827这道题，它要求如何只填一个格子后，得到一个最大岛面积，这题首先要用dfs遍历一次所有岛屿，算出面积，并为每个岛附一个标签记录它的面积，然后再去遍历所有海洋格子，看看填上这个格子之后加上前后左右（如果有）的岛屿面积是否为最大，思路比较巧妙，第一次做真想不到。想到这一点的关键在于，把思维从查看所有岛屿之间的海洋，换成查看所有海洋周围的岛屿，是一个逆向思维的过程。

```c++
unordered_map<int, int> map; // index -> area size
unordered_map<int, int> counted;
int n = 0;
int idx = 2;

bool IsInArea(int x, int y)
{
    return x >= 0 && x < n && y >= 0 && y < n;
}

int DFS_Area(vector<vector<int>>& grid, int x, int y)
{
    if (!IsInArea(x, y))
        return 0;
    if (grid[x][y] == 1)
    {
        grid[x][y] = idx;
        return 1 + 
            DFS_Area(grid, x - 1, y) + 
            DFS_Area(grid, x + 1, y) + 
            DFS_Area(grid, x, y - 1) + 
            DFS_Area(grid, x, y + 1);
    }
    return 0;
}

int GetArea(vector<vector<int>>& grid, int x, int y)
{
    if (IsInArea(x, y) && grid[x][y] != 0 && !counted.count(grid[x][y]))
    {
        counted[grid[x][y]] = 1;
        return map[grid[x][y]];
    }
    return 0;
}

int largestIsland(vector<vector<int>>& grid)
{
    n = grid.size();
    for (int i = 0; i < n; ++i)
    {
        for (int j = 0; j < n; ++j)
        {
            if (grid[i][j] == 1)
            {
                map[idx] = DFS_Area(grid, i, j);
                idx++;
            }
        }
    }
    int res = 0;
    bool temp = false;
    for (int i = 0; i < n; ++i)
    {
        for (int j = 0; j < n; ++j)
        {
            if (grid[i][j] == 0)
            {
                temp = true;
                counted.clear();
                res = max(res, 1 + 
                            GetArea(grid, i - 1, j) + 
                            GetArea(grid, i + 1, j) + 
                            GetArea(grid, i, j - 1) + 
                            GetArea(grid, i, j + 1));
            }
        }
    }
    if (!temp)
        return n * n;
    return res;
}
```

要注意的是存在没有海洋的情况，此时不需要填海直接得出面积。

其他类似的题目还有130和733，都是相似的思路。

## N皇后（[LeetCode 51]()）



