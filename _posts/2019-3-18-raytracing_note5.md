---
layout: post
title:  "光线追踪学习笔记5——折射材质"
#date:   2019-03-14 16:08:54
categories: 图形学
tags: ray_tracing
excerpt: 折射物体和镜头移动（章节9、10）
author: Tizeng
---

* content
{:toc}

## Chapter 9: Dielectrics

这一章实现半透明材质，先看一下折射定律（斯涅尔定律）：

> n1 * sinθ1 = n2 * sinθ2

### 折射函数

折射定律中`n`为介质的折射率，`θ`为入射角和折射角，规定空气的折射率是1，玻璃大概在1.3到1.7之间。下面是折射函数，其中`discriminant`其实是cosθ2，将其用折射定律和cosθ1表示时需要开根号，为了保证有解需要`discriminant`大于0，而cosθ1可以表示为`-n*v`，即法向量和入射向量（均为单位向量）的点积取负，这样一来就会得到`refracted`中的式子，即折射向量（注意也是单位向量，上面求反射光线方向向量的时候也是取的单位向量），最好在纸上推导一下:

```c++
bool refract(const vec3& v, const vec3& n, float ni_over_nt, vec3& refracted) {
    vec3 uv = unit_vector(v);
    float dt = dot(uv, n);
    float discriminant = 1.0 - ni_over_nt * ni_over_nt * (1 - dt * dt);
    if (discriminant > 0) {
        refracted = ni_over_nt * (uv - n * dt) - n * sqrt(discriminant);
        return true;
    }
    else
        return false;
}
```

### 发生折射的物体

有了计算折射光向量的函数，就可以写电介质类了（不知道作者为什么要用这个词，感觉玻璃类更好理解），成员`ref_idx`是材质的折射率，我们默认空气的折射率是1，因此若入射光线与法线的点积大于0，说明入射光线是从介质射出空气，根据折射定律，`ni_over_nt`（入射介质的折射率与材料介质折射率的比值）直接是`ref_idx`，将法线反向便于计算；反之则说明入射光线是从空气中射入介质，此时的`ni_over_nt`为`ref_idx`的倒数，法线不需要反向。

接下来调用`refract`函数，判断折射是否成功，如果成功，则将折射光线变量储存进`scattered`，如果不成功则将反射光线变量储存至`scattered`（实际上这里如果折射不成功，也不会有反射光）。衰减系数`attenuation`默认设为(1, 1, 1)。

```c++
class dielectric :public material {
public:
    dielectric(float ri) : ref_idx(ri) {}
    virtual bool scatter(const ray& ray_in, const hit_record& rec, vec3& attenuation, ray& scattered) const {
        vec3 outward_normal;
        vec3 reflected = reflect(ray_in.direction, rec.normal);
        float ni_over_nt;
        attenuation = vec3(1.0, 1.0, 1.0);
        vec3 refracted;
        if (dot(ray_in.direction(), rec.normal) > 0) {
            outward_normal = -rec.normal;
            ni_over_nt = ref_idx;
        }
        else {
            outward_normal = rec.normal;
            ni_over_nt = 1.0 / ref_idx;
        }
        if (refract(ray_in.direction(), outward_normal, ni_over_nt, refracted)) {
            scattered = ray(rec.p, refracted);
        }
        else {
            scattered = ray(rec.p, reflected);
            return false;
        }
        return true;
    }
private:
    float ref_idx;
};
```

提一个bug，在判断入射方向时有一个将法向量反向的操作`-rec.normal`，这里要做的是将其坐标值全部取反，直接在原向量上加负号，而我在重载`-`操作符实现这个操作时少打了一个负号，有一个坐标没有取反，输出图像虽然也有折射现象，但一直与正确图像不符，令人百思不得其解，看了十几次代码才想到会不会是此处出错，因此在定义这种基本类型时一定要小心，避免不必要的疏忽。

正确输出结果如下：

![sphere_glass](https://github.com/tizengyan/images/raw/master/sphere_glass.png)

用csdn上的一张图来解释左边玻璃球的图像（[原文在此](https://blog.csdn.net/libing_zeng/article/details/54428732)）：

![sphere_refract](https://github.com/tizengyan/images/raw/master/sphere_refract.png)

### 真正的介质

要注意上面的代码只考虑了折射的情况，而现实中存在全反射这一物理现象，即入射角达到一定程度时不会发生折射而只发生反射的现象，而且这是因为还存在**折射系数**，根据入射角的不同，折射系数会发生变化，它决定了有多少光被反射，多少光被折射，如果没有光发生折射，则发生全反射，要加入这一性质，玻璃材质才完整，而计算折射系数非常繁琐，图形学中经常用Schlick近似公式来模拟：

```c++
float schlick(float cosine, float ref_idx) {
    float r0 = (1 - ref_idx) / (1 + ref_idx);
    r0 = r0 * r0;
    return r0 + (1 - r0) * pow((1 - cosine), 5);
}
```

有了这个公式，就可以补全电介质（玻璃）类：

```c++
class dielectric :public material {
public:
    dielectric(float ri) : ref_idx(ri) {}
    virtual bool scatter(const ray& ray_in, const hit_record& rec, vec3& attenuation, ray& scattered) const {
        vec3 outward_normal;
        vec3 reflected = reflect(ray_in.direction(), rec.normal);
        float ni_over_nt;
        attenuation = vec3(1.0, 1.0, 1.0);
        vec3 refracted;
        float reflect_prob;
        float cosine;
        if (dot(ray_in.direction(), rec.normal) > 0) {
            outward_normal = -rec.normal;
            ni_over_nt = ref_idx;
            cosine = ref_idx * dot(ray_in.direction(), rec.normal) / ray_in.direction().length();
        }
        else {
            outward_normal = rec.normal;
            ni_over_nt = 1.0 / ref_idx;
            cosine = -dot(ray_in.direction(), rec.normal) / ray_in.direction().length();
        }
        if (refract(ray_in.direction(), outward_normal, ni_over_nt, refracted)) {
            reflect_prob = schlick(cosine, ref_idx);
        }
        else {
            scattered = ray(rec.p, reflected);
            reflect_prob = 1.0;
            //return false;
        }
        if(drand() < reflect_prob)
            scattered = ray(rec.p, reflected);
        else
            scattered = ray(rec.p, refracted);
        return true;
    }
private:
    float ref_idx;
};
```

其中`reflected`、`refracted`这两个变量长的很像，要特别小心。另外文中提到一个制作泡泡效果的小技巧，将一个球的半径设为负值，此时球面的法线会全部朝内，与另一个半径相近的同心球放在一起，就会有如下效果，如果只放半径为负的球，会出现一些奇怪的效果，具体原因不太清楚，这里就不过多提了：

```c++
list[3] = new sphere(vec3(-1, 0, -1), 0.5, new dielectric(1.5));
list[4] = new sphere(vec3(-1, 0, -1), -0.45, new dielectric(1.5));
```

效果如图：

![sphere_bubble](https://github.com/tizengyan/images/raw/master/sphere_bubble.png)

## Chapter 10: Positionable camera

* 先介绍一个计算PI的小技巧，直接调用`atan(1)*4`即可得到PI的值，`atan`函数是正切的反函数arctan，输入1会返回45度的弧度值也就是PI/4

* 衰减变量`attenuation`可以用来筛选像素的颜色通道，如果我们不需要RGB中的哪个颜色，只需将对应坐标位置的值调低甚至设为0就行了

这一章介绍可以控制**位置**和**镜头方向**的摄像机。

### 视角控制

之前我们的`camera`类是默认固定在(0,0,0)也就是原点处，视野范围是从左下角的(-2,-1,-1)开始，宽4高2的矩形窗口，我们现在对其进行扩展，在构造函数中加入两个参数`vfov`和`aspect`，前者是摄像机视野最大仰角和俯角的和（单位为度，且规定仰角与俯角相等），后者是窗口宽与高的比值。`half_height`与`half_width`即为窗口的高和宽的一半，我们规定窗口距离摄像机一个单位长度（根据需要可以变化，只要记得根据三角关系更改公式），因此有`half_height = tan(theta / 2)`，然后再由`aspect`求出`half_width`，从而得到窗口左下角坐标。

下面是新的`camera`类代码：

```c++
class camera {
public:
    camera(float vfov, float aspect) { // vfov is to to bottom in degrees
        float theta = vfov * M_PI / 180;
        float half_height = tan(theta / 2);
        float half_width = aspect * half_height;

        lower_left_corner = vec3(-half_width, -half_height, -1.0);
        horizontal = vec3(2 * half_width, 0.0, 0.0);
        vertical = vec3(0.0, 2.0 * half_height, 0.0);
        origin = vec3(0.0, 0.0, 0.0);
    }

    ray get_ray(float u, float v) {
        // 以左下角坐标为基础，根据分割的u、v值映射所有范围内的点
        return ray(origin, lower_left_corner + u * horizontal + v * vertical - origin);
    }
private:
    vec3 lower_left_corner;
    vec3 horizontal;
    vec3 vertical;
    vec3 origin;
};
```

这里我们初始化纵向视角为90度的摄像机，并摆放两个测试球：

```c++
camera cam(90, float(nx) / float(ny));
...
float R = cos(M_PI / 4);
list[0] = new sphere(vec3(-R, 0, -1), R, new lambertian(vec3(0, 0, 1)));
list[1] = new sphere(vec3( R, 0, -1), R, new lambertian(vec3(1, 0, 0)));
```

效果如下：

![new_camera_test](https://github.com/tizengyan/images/raw/master/new_camera_test.png)

### 摄像机位置控制

此时的摄像机便已经有了设置视角的功能，但它仍被定死在原点，为了能自如的移动摄像机，我们需要引入几个向量：

![camera_vectors](https://github.com/tizengyan/images/raw/master/camera_vectors.png)

上图右边相当于是摄像机的右视图，虚线就是摄像机的**视线**。

首先作为一个摄像机，需要一个位置坐标`lookfrom`，和一个镜头方向`lookat`，这两个向量决定了摄像机的位置和朝向，`u`和`v`为摄像机所在平面（经过`lookfrom`且与摄像机视线垂直的平面）的单位横、纵向量，`w`为与视线相反的单位向量，也就是说，摄像机视野的朝向`unit(lookat - lookfrom) = -w`。而`vup`是一个指向上方的向量，输入时规定它的坐标为(0,1,0)，利用向量的叉乘可以很容易的用它求出其他向量。

下面是完整的`camera`类：

```c++
class camera {
public:
    camera(vec3 lookfrom, vec3 lookat, vec3 vup, float vfov, float aspect) { // vfov is to to bottom in degrees
        vec3 u, w, v;
        float theta = vfov * M_PI / 180;
        float half_height = tan(theta / 2);
        float half_width = aspect * half_height;
        origin = lookfrom;
        w = unit_vector(lookfrom - lookat);
        u = unit_vector(cross(vup, w));
        v = cross(w, u);

        lower_left_corner = vec3(-half_width, -half_height, -1.0);
        lower_left_corner = origin - half_height * v - half_width * u - w;
        //horizontal = vec3(2 * half_width, 0.0, 0.0);
        //vertical = vec3(0.0, 2.0 * half_height, 0.0);
        horizontal = 2 * half_width * u;
        vertical = 2 * half_height * v;
    }

    ray get_ray(float s, float t) {
        // 以左下角坐标为基础，根据分割的u、v值映射所有范围内的点
        return ray(origin, lower_left_corner + s * horizontal + t * vertical - origin); 
    }
private:
    vec3 lower_left_corner;
    vec3 horizontal;
    vec3 vertical;
    vec3 origin;
};
```

于是我们就可以在`main`中初始化不同位置的摄像机，来对场景的物体测试，比如：

```c++
camera cam(vec3(-2, 2, 1), vec3(0, 0, -1), vec3(0, 1, 0), 90, float(nx) / float(ny));
```

这个位置的摄像机会得到如下视野：

![cam_view1](https://github.com/tizengyan/images/raw/master/cam_view1.png)

通过更改位置向量，我们现在可以随心所欲的操纵摄像机了！

![cam_view2](https://github.com/tizengyan/images/raw/master/cam_view2.png)