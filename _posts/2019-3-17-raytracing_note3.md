---
layout: post
title:  "光线追踪学习笔记3"
#date:   2019-03-14 16:08:54
categories: 图形学
tags: ray_tracing
excerpt: 开始画球
author: Tizeng
---

* content
{:toc}

前面定义了`vec3`这个类来储存三维向量的空间坐标，但它其实还有另一个用处，就是储存彩色像素点的RGB值，它同样是三维的，因此看到`vec3`定义的变量即有可能是空间点的坐标，又有可能是像素值，注意不要混淆。

## Chapter 4: Adding a sphere

类`ray`中有一个参数`t`，它不是其中的成员变量，因为起始点和照射方向两点确定了一条直线（或者说一个向量），`t`是用来衡量这束光线途径位置的变量，可以理解为“时间”，即发出光线后经过时间`t`后所到达的位置点。

那么我们可以用`t`作为一个参数，来判断光线是否到达了球体。在三维空间中球体的方程为：

> (x-cx) * (x-cx) + (y-cy) * (y-cy) + (z-cz) * (z-cz) = R * R

圆心为(cx, cy, cz)。我们把它记为向量`C`，把空间中任意一点(x, y, z)记为`p`，则上面的方程可以用向量的点乘来表示：

> dot( (p - C), (p - C) ) = R*R

由于光线在之前的定义中用向量表示为`A + t * B`，因此我们可以得到落在球体的光线的向量关系：

> dot( (A + t * B - C), (A + t * B - C) ) = R * R

根据点积的性质，我们很容易可以将其写为如下形式：

> t * t * dot(B, B) + 2t * dot(B, A - C) + dot(A - C, A - C) - R*R = 0

那么现在就得到了一个`t`的一元二次方程方程，`A`、`B`、`C`皆已知，通过计算`delta`，可以得知此时光线是否到达了球体。

```c++
bool hit_sphere(const vec3& center, float radius, const ray& r) {
    vec3 oc = r.origin() - center;
    float a = dot(r.direction(), r.direction());
    float b = 2.0 * dot(r.direction(), oc);
    float c = dot(oc, oc) - radius * radius;
    float delta = b * b - 4 * a *c;
    return delta > 0;
}

vec3 color(const ray& r) {
    // return RGB values using vec3
    if (hit_sphere(vec3(0, 0, -1), 0.5, r)) {
        return vec3(1, 0, 0);
    }
    vec3 unit_direction = unit_vector(r.direction());
    float t = 0.5 * (unit_direction.y() + 1.0);
    return (1.0 - t) * vec3(1.0, 1.0, 1.0) + t * vec3(0.5, 0.7, 1.0);
}
```

`main`函数不变，上色函数`color`加入对球的填色代码，通过调用`hit_sphere`函数，判断相应的像素点是否应该显示球的颜色。在加入其他代码之前这个球的颜色根据`color`函数中给定的颜色来决定。

## Chapter 5: Surface normals and multiple objects

这里 normal 表示法线，为了加入阴影，我们需要得到球面的法线坐标。这里我们定义法线的长度始终为单位长度，方向向外，具体来说就是球面点的向量减去球心向量。

在加入更复杂的阴影系统前，我们用色盘来显示法线信息，若光线和球有交点，先计算该法线的方向并将其单位化，此时各个坐标的范围落在[-1, 1]，为了将它们直接映射成RGB值，需要先将范围改到[0, 1]:

```c++
float hit_sphere(const vec3& center, float radius, const ray& r) {
    vec3 oc = r.origin() - center;
    float a = dot(r.direction(), r.direction());
    float b = 2.0 * dot(r.direction(), oc);
    float c = dot(oc, oc) - radius * radius;
    float delta = b * b - 4 * a *c;
    if (delta >= 0)
        return (-b - sqrt(delta)) / (2.0 * a);
    else
        return -1;
}

vec3 color(const ray& r) {
    // return RGB values using vec3
    float t =  hit_sphere(vec3(0, 0, -1), 0.5, r);
    if (t > 0)) {
        ve3 N = unit_vector(r.point_at_parameter(t) - vec3(0, 0, -1));
        return 0.5 * vec3(N.x() + 1, N.y() + 1, N.z() + 1);
    }
    vec3 unit_direction = unit_vector(r.direction());
    float t = 0.5 * (unit_direction.y() + 1.0);
    return (1.0 - t) * vec3(1.0, 1.0, 1.0) + t * vec3(0.5, 0.7, 1.0);
}
```

输出结果如下：

![sphere1](https://github.com/tizengyan/images/raw/master/sphere1.png)

### 多个物体

如果我们需要多个物体都显示在摄像机的范围内，就需要创建一个抽象类`hitable`，任何需要显示的类都继承自这个类，还需要一个结构体来储存法线和坐标信息。

```c++
struct hit_record {
    float t;
    vec3 p;
    vec3 normal;
};

class hitable {
public:
    virtual bool hit(const ray& r, float t_min, float t_max, hit_record& rec) const = 0;
};
```

接下来我们将之前的代码修改为显示的球继承自`hitable`的实现方法，每个子类都要重写`hit`函数，用以之后的着色判断，在方程有解，也就是坐标在球上时，判断其`t`是否在[t_min, t_max]范围内，如果在，那么更新法线信息等至`rec`，返回`true`：

```c++
class sphere :public hitable {
public:
    sphere() {}
    sphere(vec3 cen, float t): center(cen), radius(t) {};
    virtual bool hit(const ray& r, float tmin, float tmax, hit_record& rec) const;
private:
    vec3 center;
    float radius;
};

bool sphere::hit(const ray& r, float t_min, float t_max, hit_record& rec) const {
    vec3 oc = r.origin() - center;
    float a = dot(r.direction(), r.direction());
    float b = dot(r.direction(), oc);
    float c = dot(oc, oc) - radius * radius;
    float delta = b * b - a * c;
    if (delta > 0) {
        float temp = (-b - sqrt(b*b - a * c)) / a;
        if (temp < t_max && temp > t_min) {
            rec.t = temp;
            rec.p = r.point_at_parameter(rec.t);
            rec.normal = (rec.p - center) / radius;
            return true;
        }
        temp = (-b + sqrt(b*b - a * c)) / a;
        if (temp < t_max && temp > t_min) {
            rec.t = temp;
            rec.p = r.point_at_parameter(rec.t);
            rec.normal = (rec.p - center) / radius;
            return true;
        }
    }
    return false;
}
```

除此之外还需要一个储存当前需显示物体信息的类`hitable_list`，它的主要功能除了存储对象之外，还在重写的`hit`函数中，通过更新一个最小值`cloest_so_far`，在储存的所有物体中，计算离摄像机最近（`t`最小）的物体是哪一个对象，同时调用对象本身的`hit`函数：

```c++
class hitable_list :public hitable {
public:
    hitable_list() {}
    hitable_list(hitable **l, int n) : list(l), list_size(n) {};
    virtual bool hit(const ray& r, float tmin, float tmax, hit_record& rec) const;
private:
    hitable **list;
    int list_size;
};

bool hitable_list::hit(const ray& r, float t_min, float t_max, hit_record& rec) const {
    hit_record temp_rec;
    bool is_hit = false;
    double cloest_so_far = t_max;
    for (int i = 0; i < list_size; i++) {
        if (list[i]->hit(r, t_min, cloest_so_far, temp_rec)) {
            is_hit = true;
            cloest_so_far = temp_rec.t;
            rec = temp_rec;
        }
    }
    return is_hit;
}
```

最后是修改过的主函数和着色函数，在着色前先建立两个球体，并保存信息至一个`hitable_list`对象`world`，挨个对像素点着色时，调用`color`函数，此时接受的变量是光线`r`和视野中的物体列表`world`，对其调用`hit`，只要其不为空就会返回`true`，并将物体的颜色等信息保存至`rec`，最后调用`rec`中的法线信息对其进行着色：

```c++
vec3 color(const ray& r, hitable *world) {
    // return RGB values using vec3
    hit_record rec;
    if (world->hit(r, 0.0, FLT_MAX, rec)) {
        return 0.5 * vec3(rec.normal.x() + 1, rec.normal.y() + 1, rec.normal.z() + 1);
    }
    else {
        vec3 unit_direction = unit_vector(r.direction());
        float t = 0.5 * (unit_direction.y() + 1.0);
        return (1.0 - t) * vec3(1.0, 1.0, 1.0) + t * vec3(0.5, 0.7, 1.0);
    }
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
    hitable *list[2];
    list[0] = new sphere(vec3(0, 0, -1), 0.5);
    list[1] = new sphere(vec3(0, -100.5, -1), 100);
    hitable_list *world = new hitable_list(list, 2);
    for (int j = ny - 1; j >= 0; j--) {
        for (int i = 0; i < nx; i++) {
            float u = float(i) / float(nx);
            float v = float(j) / float(ny);
            ray r(origin, lower_left_corner + u * horizontal + v * vertical);

            vec3 p = r.point_at_parameter(2.0);
            vec3 col = color(r, world);
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

修改后的输出结果如下，可以看到屏幕中有上下两个一大一小的球体：

![sphere2](https://github.com/tizengyan/images/raw/master/sphere2.png)