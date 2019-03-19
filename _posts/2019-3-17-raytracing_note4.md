---
layout: post
title:  "光线追踪学习笔记4——材质"
#date:   2019-03-14 16:08:54
categories: 图形学
tags: ray_tracing
excerpt: 加特技（章节6、7、8）
author: Tizeng
---

* content
{:toc}

## Chapter 6: Antialiasing

这一章告诉我们如何把物体的边缘和环境融合的更好，而不是单纯又突兀的颜色变换。

创建`camera`类，和一个可以随机出落在[0, 1)范围内随机数的函数，在每次着色时对像素周围的其他像素进行取样，取样数量定为`ns=100`，着色前将所有样本的像素值平均，得到当前像素值，这样输出的图像在边缘就会有模糊的效果。

```c++
class camera {
public:
    camera() {
        lower_left_corner = vec3(-2.0, -1.0, -1.0);
        horizontal = vec3(4.0, 0.0, 0.0);
        vertical = vec3(0.0, 2.0, 0.0);
        origin = vec3(0.0, 0.0, 0.0);
    }

    ray get_ray(float u, float v) {
        // 以左下角坐标为基础，根据分割的u、v值映射所有范围内的点
        return ray(origin, lower_left_corner + u * horizontal + v * vertical);
    }
private:
    vec3 lower_left_corner;
    vec3 horizontal;
    vec3 vertical;
    vec3 origin;
};
```

还需要一个返回[0,1)随机数的函数，用于对像素周围的点取样：

```c++
float drand() {
    return (float)rand() / (RAND_MAX + 1);
}
```

有了摄像机类，我们就可以在主函数中创建其对象，实现之前的效果，可以看到此时物体的边缘更好的与环境融合：

```c++
int ns = 100;
camera cam;
...
for (int s = 0; s < ns; s++) {
    float u = float(i + drand()) / float(nx);
    float v = float(j + drand()) / float(ny);

    ray r = cam.get_ray(u, v);
    vec3 p = r.point_at_parameter(2.0);
    col += color(r, world);
}
col /= float(ns);
...
```

输出图像：

![sphere_blur](https://github.com/tizengyan/images/raw/master/sphere_blur.png)

## Chapter 7: Diffuse Materials

本章实现一个漫反射物体的效果。

### 漫反射模型

漫反射表面不会发出光线，而是将射过来的光线反射至随机方向，并根据自身的颜色做出调整，因此若光线射入两个平面的缝隙，在这两个平面上会各自发生随机的漫反射，这很容易理解。下面这张示意图有点容易产生误解，要结合原文描述仔细揣摩：

![tangent_sphere](https://github.com/tizengyan/images/raw/master/tangent_sphere.png)

下面的平面是我们的物体平面，那个点是光线的接触点`p`，现在以这个点相切做一个单位球，并在单位球上随机取一个点`s`，这便是点`p`漫反射后的光线方向，要得到这个点的向量并不难，首先`random_in_unit_sphere`函数给出单位圆圆心到`s`的向量，实现方法是很简单的拒绝法，随机在与单位圆相切的单位立方体中取一点，判断其是否在单位圆内，如果是则返回，不是则重新取。

我们现在只需要原点到单位圆的向量了，这个可以通过`p + N`得到，其中`N`为通过`p`点的法线向量。

下面是新的着色代码：

```c++
vec3 random_in_unit_sphere() {
    vec3 p;
    do {
        p = 2.0 * vec3(drand(), drand(), drand()) - vec3(1, 1, 1);
    } while (p.squared_length() >= 1);
    return p;
}

vec3 color(const ray& r, hitable *world) {
    // return RGB values using vec3
    hit_record rec;
    if (world->hit(r, 0.0, FLT_MAX, rec)) {
        vec3 target = rec.p + rec.normal + random_in_unit_sphere();
        return 0.5 * color(ray(rec.p, target - rec.p), world);
    }
    else {
        vec3 unit_direction = unit_vector(r.direction());
        float t = 0.5 * (unit_direction.y() + 1.0);
        return (1.0 - t) * vec3(1.0, 1.0, 1.0) + t * vec3(0.5, 0.7, 1.0);
    }
}
```

这里用一个递归的调用，来计算重复反射的情况，新的光线起始点存在`rec.p`中，方向为`rec.p`指向`target`的向量

如果球的颜色太深，只需对主函数作如下修改，将RGB的值在映射之前先开根号以增加亮度：

```c++
col /= float(ns);
col = vec3(sqrt(col[0]), sqrt(col[1]), sqrt(col[2]));
int ir = int(255.99 * col[0]);
int ig = int(255.99 * col[1]);
int ib = int(255.99 * col[2]);
```

效果如下所示：

![sphere_shade](https://github.com/tizengyan/images/raw/master/sphere_shade.png)

## Chapter 8: Metal

对于金属而言，光线的反射相当于镜面反射，入射角等于反射角，因此可以得出反射向量等于入射向量`v`加上两倍的`B`，要得到`B`则需要`v`与法线`N`的点乘取负，因为夹角大于九十度。

![reflect_feature](https://github.com/tizengyan/images/raw/master/reflect_feature.png)

所以可以写出得到反射向量的函数：

```c++
inline vec3 reflect(const vec3& v, const vec3& n) {
    return v - 2 * dot(v, n) * n;
}
```

我们可能需要多种材质，这时可以先建立一个抽象类`material`，然后根据不同材质的需要去实现散射函数，比如我们先实现一个漫反射性质的材料和一个金属质的材料：

```c++
class material {
public:
    virtual bool scatter(const ray& ray_in, const hit_record& rec, vec3& attenuation, ray& scattered) const = 0;
};

class lambertian :public material {
private:
    vec3 albedo;
public:
    lambertian(const vec3& a) : albedo(a) {}
    virtual bool scatter(const ray& ray_in, const hit_record& rec, vec3& attenuation, ray& scattered) const {
        vec3 target = rec.p + rec.normal + random_in_unit_sphere();
        scattered = ray(rec.p, target - rec.p);
        attenuation = albedo;
        return true;
    }
};

class metal :public material {
public:
    metal(const vec3& a): albedo(a) {}
    virtual bool scatter(const ray& ray_in, const hit_record& rec, vec3& attenuation, ray& scattered) const {
        vec3 reflected = reflect(unit_vector(ray_in.direction()), rec.normal);
        scattered = ray(rec.p, reflected);
        attenuation = albedo;
        return dot(scattered.direction(), rec.normal) > 0;
    }
private:
    vec3 albedo;
};
```

这里漫反射材质和上一章的实现一样，反射的光线向量为反射点向量`p`加上法线向量`N`，即是（从原点出发）指向与`p`点相切的单位球球心的向量，再加上球内随机的一点，就是反射过后的光线向量坐标，记为`target`，要得到它的光线表示，即用`ray`类型表示，它的起始点是`p`，方向向量为`target - p`，然后将得出的反射光线变量保存至`scattered`。

对于金属材质来说，我们首先要获得反射后的向量`reflected`，在这之前我们需要获得入射向量并将其单位化，然后与入射点的法线向量一同带入之前的`reflect`函数。得到反射向量后，便可以将其保存进`scattered`从而得到反射后的光线变量。

### 对不同材料着色

变量`attenuation`就是材质在三个维度的衰减程度，它会影响后面着色时取到的像素值的高低。每次射入光线时都要更新这个值，告诉外部我应该以什么样的反射率来计算像素值。另外，如果反射角大于九十度，说明光线被物体吸收，不做反射，漫反射材质没有做这个判断是因为我们始终在表面的一个单位球中寻找反射向量，因此始终返回`true`。

有了新的材质类后，在`hit_record`结构中增加一个新成员`mat_ptr`用以记录材质类型，就可以修改着色函数了，这里是一个多态的应用，我们把光线的反射在材质中的`scatter`函数中实现，着色函数中只需要调用储存的`rec`信息，直接调用`scatter`，就可以运行目标材质中的实现。深度`depth`是一个控制递归深度的变量，防止计算过多的情况。

```c++
vec3 color(const ray& r, hitable *world, int depth) {
    hit_record rec;
    if (world->hit(r, 0.001, FLT_MAX, rec)) {
        ray scattered;
        vec3 attenuation;
        if (depth < 50 && rec.mat_ptr->scatter(r, rec, attenuation, scattered)) {
            return attenuation * color(scattered, world, depth + 1);
        }
        else {
            return vec3(0, 0, 0);
        }
    }
    else {
        vec3 unit_direction = unit_vector(r.direction());
        float t = 0.5 * (unit_direction.y() + 1.0);
        return (1.0 - t) * vec3(1.0, 1.0, 1.0) + t * vec3(0.5, 0.7, 1.0);
    }
}
```

同样的，在球类中我们需要加入一个`material`的指针成员，来记录自身的材质，然后在主程序中作如下改动：

```c++
    hitable *list[4];
    list[0] = new sphere(vec3(0, 0, -1), 0.5, new lambertian(vec3(0.8, 0.3, 0.3)));
    list[1] = new sphere(vec3(0, -100.5, -1), 100, new lambertian(vec3(0.8, 0.8, 0.0)));
    list[2] = new sphere(vec3( 1, 0, -1), 0.5, new metal(vec3(0.8,0.6,0.2))); // 金属球
    list[3] = new sphere(vec3(-1, 0, -1), 0.5, new metal(vec3(0.8,0.8,0.8))); // 金属球
    hitable_list *world = new hitable_list(list, 4);
```

输出结果：

![sphere_metal](https://github.com/tizengyan/images/raw/master/sphere_metal.png)

### 模糊效果

对于金属材质，我们可以给它增加一个模糊效果，方法很简单，即在反射光线向量顶端做一个单位球，让它随机的在这个球内进行偏移，具体看下图：

![fuzz](https://github.com/tizengyan/images/raw/master/fuzz.png)

如果追求更高的模糊效果，可以把这个球加大。接着在`metal`类中加入一个模糊系数变量，在给`scatter`保存光线变量前，给`reflected`向量做一个偏移，就可以了：

```c++
class metal :public material {
public:
    metal(const vec3& a, float f) : albedo(a) { if (f < 1) fuzz = f; else fuzz = 1; }
    virtual bool scatter(const ray& ray_in, const hit_record& rec, vec3& attenuation, ray& scattered) const {
        vec3 reflected = reflect(unit_vector(ray_in.direction()), rec.normal);
        scattered = ray(rec.p, reflected + fuzz * random_in_unit_sphere()); // 此处进行了模糊
        attenuation = albedo;
        return dot(scattered.direction(), rec.normal) > 0;
    }
private:
    vec3 albedo;
    float fuzz;
};
```

两边为模糊系数为0.1（左）和0.3（右）的球，注意模糊系数越大模糊效果越强，如果不需要模糊，初始化材质时设`fuzz`为0即可：

![sphere_with_fuzz](https://github.com/tizengyan/images/raw/master/sphere_with_fuzz.png)