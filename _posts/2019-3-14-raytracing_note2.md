---
layout: post
title:  "光线追踪学习笔记2——光线"
#date:   2019-03-14 16:08:54
categories: 图形学
tags: ray_tracing
excerpt: 开始制作光线（章节3）
author: Tizeng
---

* content
{:toc}

## Chapter 3: Rays, a simple camera, and background

定义一个光线类`ray`，从点`A`照向点`B`，根据输入参数`t`控制长度。将屏幕按通常习惯分为xy轴，建立右手坐标系，此时z轴的正向应垂直于屏幕射向我们。

```c++
#ifndef RAYH
#define RAYH
#include "vec3.h"

class ray {
private:
    vec3 A;
    vec3 B;
public:
    ray(){}
    ray(const vec3& a, const vec3 &b) { A = a; B = b; }
    vec3 origin() { return A; }
    vec3 direction() { return B; }
    vec3 point_at_parameter(float t) const { return A + B * t; }
};
#endif // !RAYH
```

主程序中两个`for`循环是从图像的左上角开始，逐行扫描200x100所有像素点，先左右后，然后先上后下，`u`和`v`就是每次扫描像素点的横纵坐标，但这是基于(0, 0, -1)来的，这是画面中的原点，我们实际的三维的原点（也就是摄像头）距离画面一个单位长度，而整个画面的起始点应该是在左下角`lower_left_corner`(-2, -1, -1)处，因此需要做一个映射。

![camera](https://github.com/tizengyan/images/raw/master/camera_view.png)


这里要介绍一个新概念：插值和线性插值（linear interpolation）。

首先看一下维基对插值的定义，数学的数值分析领域中，内插或称插值（英语：interpolation）是一种通过已知的、离散的数据点，在范围内推求新数据点的过程或方法。求解科学和工程的问题时，通常有许多数据点借由采样、实验等方法获得，这些数据可能代表了有限个数值函数，其中自变量的值。而根据这些数据，我们往往希望得到一个连续的函数（也就是曲线）；或者更密集的离散方程与已知数据互相吻合，这个过程叫做拟合。

光线`r`有两个向量属性，分别是起始位置`A`和照射方向`B`，初始化时我们将摄像头的位置作为起始位置（`origin`），屏幕上的点作为照射方向（`direction`），可以理解为`r`就是我们的视线方向，再将`r`输入`color`函数。这个函数的目的是通过视线坐标的`y`的位置来  ，`y`本来根据我们建立的坐标系范围会落在[-1, 1]，我们为了方便将它重新scale在`t`上，`t`属于[0, 1]，然后根据`t`的值分部红色和绿色，而蓝色始终给满，就会得出那张视线下方纯白，上方淡蓝的效果图。
`(1.0 - t) * vec3(1.0, 1.0, 1.0) + t * vec3(0.5, 0.7, 1.0)`这一行就是在做线性插值，说白了就是均匀的做了个映射。

下面是主程序：

```c++
vec3 color(const ray& r) {
    vec3 unit_direction = unit_vector(r.direction());
    float t = 0.5 * (unit_direction.y() + 1.0);
    return (1.0 - t) * vec3(1.0, 1.0, 1.0) + t * vec3(0.5, 0.7, 1.0);
}

int main() {
    int nx = 200;
    int ny = 100;
    fstream fs;
    fs.open("test.ppm", std::fstream::in | std::fstream::out | fstream::trunc);
    //cout << "P3\n" << nx << " " << ny << "\n255\n";
    fs << "P3\n" << nx << " " << ny << "\n255\n";
    vec3 lower_left_corner(-2.0, -1.0, -1.0);
    vec3 horizontal(4.0, 0.0, 0.0);
    vec3 vertical(0.0, 2.0, 0.0);
    vec3 origin(0.0, 0.0, 0.0);
    for (int j = ny - 1; j >= 0; j--) {
        for (int i = 0; i < nx; i++) {
            //vec3 v(float(i) / float(nx), float(j) / float(ny), 0.2);
            float u = float(i) / float(nx);
            float v = float(j) / float(ny);
            ray r(origin, lower_left_corner + u * horizontal + v * vertical);
            vec3 col = color(r);
            int ir = int(255.99 * col[0]);
            int ig = int(255.99 * col[1]);
            int ib = int(255.99 * col[2]);
            //cout << ir << " " << ig << " " << ib << "\n";
            fs << ir << " " << ig << " " << ib << "\n";
        }
    }
    fs.close();
    return 0;
}
```

输出图像：

![output](https://github.com/tizengyan/images/raw/master/output3.png)