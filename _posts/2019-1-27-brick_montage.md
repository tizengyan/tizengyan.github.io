---
layout: post
title:  "笔试题：花砖拼接"
date:   2019-01-27 11:11:54
categories: 算法题
tags: quiz
excerpt: 网易笔试题回顾
author: Tizeng
---

题目概述：

![brick quiz](https://github.com/tizengyan/images/raw/master/brick1.png){:height="70%" width="70%"}
![brick quiz](https://github.com/tizengyan/images/raw/master/brick2.png){:height="70%" width="70%"}
![brick quiz](https://github.com/tizengyan/images/raw/master/brick3.png){:height="70%" width="70%"}
![brick quiz](https://github.com/tizengyan/images/raw/master/brick4.png){:height="70%" width="70%"}

## 思路

这题其实不难，重要的是理清楚思路。首先我们有长度为N的砖块，且N为奇数，在一个边长为M（M>N）的墙上要将砖块完全对称的摆上去，其实就是先对称的用砖块摆好刚好覆盖MxM的区域后，再把边缘处多余的部分裁减掉。代码中则可以先如此生成一个矩阵'q'，再将其中MxM区域的图案一一赋值到一个新的MxM矩阵就行了。

## 代码实现

```c++
vector<vector<char> > f(vector<vector<char> > z, int n, int m){
    int time = 1;
    for(; time*n < m; time += 2); // 每行、列中需要砖块的个数（包括不完整砖块）
    int max = time * n;
    // 得到 time * N 大小的矩阵q
    vector<vector<char> > q;
    for(int i = 0; i < n; i++){
        vector<char> t;
        for(int j = 0; j < time; j++){
            t.insert(t.end(), z[i].begin(), z[i].end());
        }
        q.push_back(t);
    }
    for(int i = 1; i < time; i++){
        for(int j = 0; j < n ; j++)
            q.push_back(q[j]);
    }
    vector<char> t(m);
    vector<vector<char> > result(m, t);
    int x = (max-m)/2;
    // 将q正中间MxM区域的值一一赋值到新的result矩阵
    for(int i = 0; i < m; i++,x++){
        int y = (max-m) / 2;
        for(int j = 0; j < m; j++, y++){
            result[i][j] = q[x][y];
        }
    }
    return result;
}

int main(){
    int n, m;
    scanf("%d %d", &n, &m);
    vector<vector<char> > vn;
    char* p = new char[n];
    for (int i = 0; i < n; i++){
        scanf("%s", p);
        vector<char> t;
        for (int j = 0; j < n; j++){
            t.push_back(p[j]);
        }
        vn.push_back(t);
    }
    vector<vector<char> > vm = f(vn, n, m);
    for (int i = 0; i < m; i++){
        for (int j = 0 ; j < m;j++){
            printf("%c", vm[i][j]);
        }
        printf("\n");
    }
    system("pause");
    return 0;
}
```
