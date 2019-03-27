---
layout: post
title:  "光线追踪学习笔记6——景深"
#date:   2019-03-14 16:08:54
categories: 图形学
tags: ray_tracing
excerpt: 景深和最后的实验成品（章节11、12）
author: Tizeng
---

* content
{:toc}

## Chapter 11: Defocus Blur

这一章直译是离焦模糊，但作者自己在文章承认其实讲的就是景深（depth of field）。

### 什么是景深

指的是相机对焦点前后相对清晰的成像范围，在景深之内的影像比较清楚，在这个范围之前或是之后的影像则比较模糊。虽然透镜只能够将光聚到某一固定的距离，远离此点则会逐渐模糊，但是在某一段特定的距离内，影像模糊的程度是肉眼无法察觉的，这段距离称之为景深（wiki）。

### 聚焦

我们之前创建的摄像机与现实的摄像机还有一点区别，便是我们的`camera`类是从一个点中发出光线（虽然这不符合物理定律），照射到场景中的各个物体，但现实中的摄像机镜头其实是一个**透镜**，它有自己的半径和大小，而且会**聚焦**：

![cam_lens](https://github.com/tizengyan/images/raw/master/cam_lens.png)

我们在模拟的时候可以加入这一性质，进一步完善`camera`类，首先需要一个随机函数，返回xy平面内在半径为1的圆中的点：

```c++
vec3 random_in_unit_disk() {
    vec3 p;
    do {
        p = 2 * vec3(drand(), drand(), 0) - vec3(1, 1, 0);
    } while (dot(p, p) >= 1);
    return p;
}
```

这个函数的目的是模拟圆形的透镜，其上任意一点都可能接受（发出）光线，我们通过这个函数来修改`camera`中的`get_ray`函数，得到随机点的坐标后，还要根据透镜的半径、所在平面做映射，得到的`offset`便是我们的原始摄像机中心，也就是透镜的圆心往圆内偏移的量，用它对之前的光线变量做相应的偏移，再返回，这样摄像机发出的光线就是透镜上随机的点了，而`lookat`没有变化，因此所有光线都会聚集在焦点处，就像上面的图一样。

下面是新的`camera`代码：

```c++
class camera {
public:
    camera(vec3 lookfrom, vec3 lookat, vec3 vup, float vfov, float aspect, float aperture, float focus_dist) { // vfov is to to bottom in degrees
        lens_radius = aperture / 2;
        float theta = vfov * M_PI / 180;
        float half_height = tan(theta / 2);
        float half_width = aspect * half_height;
        origin = lookfrom;
        w = unit_vector(lookfrom - lookat);
        u = unit_vector(cross(vup, w));
        v = cross(w, u);

        lower_left_corner = origin - half_height * focus_dist * v  - half_width * focus_dist * u - w * focus_dist;
        horizontal = 2 * half_width * focus_dist * u;
        vertical = 2 * half_height * focus_dist * v;
    }

    ray get_ray(float s, float t) {
        vec3 rd = random_in_unit_disk() * lens_radius;
        vec3 offset = rd.x() * u + rd.y() * v;
        // 以左下角坐标为基础，根据分割的u、v值映射所有范围内的点
        return ray(origin + offset, lower_left_corner + s * horizontal + t * vertical - origin - offset);
    }
private:
    vec3 lower_left_corner;
    vec3 horizontal;
    vec3 vertical;
    vec3 origin;
    vec3 u, w, v;
    float lens_radius;
};
```

注意在摄像机的三个基本向量`lower_left_corner`、`horizontal`、`vertical`中都根据焦距调整了长度，因为我们此时我们已经不再规定摄像机离窗口是一个单位长度了，这个非常关键，不然一旦焦距不是1，摄像机就可能什么也拍不到。

`aperture`为光圈直径，`focus_dist`为焦距，即`lookfrom`与`lookat`的距离，这些可以在主函数中给定：

```c++
vec3 lookfrom(3, 3, 2);
vec3 lookat(0, 0, -1);
float dist_to_focus = (lookat - lookfrom).length();
float aperture = 2.0;
camera cam(lookfrom, lookat, vec3(0, 1, 0), 20, float(nx) / float(ny), aperture, dist_to_focus);
```

以下是光圈直径为2.0、1.0、0.5时的图像效果：

![cam_with_aperture2.0](https://github.com/tizengyan/images/raw/master/cam_with_aperture2.png)

![cam_with_aperture1.0](https://github.com/tizengyan/images/raw/master/cam_with_aperture1.png)

![cam_with_aperture0.5](https://github.com/tizengyan/images/raw/master/cam_with_aperture05.png)

可以看到光圈越小，图像越清晰。

## Chapter 12: Where next?

终于到最后一章了，真是可喜可贺！

这一章应用之前讲的系统，随机生成一个有许多不同材质的球的场景，其中有玻璃材质、金属材质和粗糙（漫反射）材质，函数`random_scene`就是为了实现这个功能，它返回一个`hitable_list`，其中包含了随机生成的众多球，再由主函数来着色：

```c++
hitable *random_scene() {
    int n = 150;
    hitable **list = new hitable*[n + 1];
    list[0] = new sphere(vec3(0, -1000, 0), 1000, new lambertian(vec3(0.5, 0.5, 0.5))); // 生成平面（其实就是一个巨大的球）
    int i = 1;
    for (int a = -11; a < 11; a = a + 2) {
        for (int b = -11; b < 11; b = b + 2) {
            float choose_mat = drand();
            vec3 center(a + 0.9 * drand(), 0.2, b + 0.9 * drand());
            if ((center - vec3(4,2,0)).length() > 0.9) {
                if (choose_mat < 0.8) { // diffuse
                    list[i++] = new sphere(center, 0.2, new lambertian(vec3(drand() * drand(), drand() * drand(), drand() * drand())));
                }
                else if (choose_mat < 0.95) { // metal
                    list[i++] = new sphere(center, 0.2, 
                        new metal(vec3(0.5 * (1 + drand()), 0.5 * (1 + drand()), 0.5 * (1 + drand())), 0.5 * drand()));
                }
                else { // glass
                    list[i++] = new sphere(center, 0.2, new dielectric(1.5));
                }
            }
        }
    }

    list[i++] = new sphere(vec3(0, 1, 0), 1.0, new dielectric(1.5));
    list[i++] = new sphere(vec3(-4, 1, 0), 1.0, new lambertian(vec3(0.4, 0.2, 0.1)));
    list[i++] = new sphere(vec3(4, 1, 0), 1.0, new metal(vec3(0.7, 0.6, 0.5), 0.0));

    return new hitable_list(list, i);
}
```

首先生成一个半径很大的球作为地面，其他的球散落在上面，当这个球足够大时，近似可以被当成一个平面，因此只要将其他小球的纵坐标y设为自己的半径就行了。

为了让小球的位置尽可能的随机，我们在循环的时候可以利用一下循环变量，让小球均匀分布在摄像机的视角内。每次生成球之前取一个随机数`choose_mat`，用来选择球的材质，在条件语句中通过阈值来控制不同材质出现的比例。最后生成三个不同材质的稍大一点的球，放在场景中间。

另外还可以利用镜头朝向和光圈大小来对焦，实现背景虚化，但光圈不宜过大，否则画面会非常模糊，下面是光圈为0.2，分辨率为640x360时的效果图，此时对焦的是最右边的大金属球，可以看到除了对焦对象之外其余的物体都比较模糊：

![random_scene1](https://github.com/tizengyan/images/raw/master/random_scene1.ppm)

再试一下光圈为0.07时的效果，调整摄像机位置并增加分辨率到720p：

![random_scene3](https://github.com/tizengyan/images/raw/master/random_scene3.png)

