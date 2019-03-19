---
layout: post
title:  "光线追踪学习笔记1——基本类和图像"
#date:   2019-03-14 16:08:54
categories: 图形学
tags: ray_tracing
excerpt: 以ppm格式输出图像，创建基本类vec3（章节1、2）
author: Tizeng
---

* content
{:toc}

## 电子书《Ray tracing in one weekend》读书笔记

### Chapter 1: Output an image

这里建立一个用于储存三维向量信息的类`vec3`，并重载了所需的各种运算符。

生成一个`.ppm`图形文件，这里可以用`cout`打印计算出的的RGB像素值然后将`.exe`文件在`cmd`命令行用`>`操作符输出到新的`.ppm`文件，也可以直接用文件流`fstream`，创建一个新的`.ppm`文件并用`<<`操作符写入像素值等信息，最后不要忘了用`close()`关闭文件流。

主程序是根据给定的横纵像素数量，将各个颜色的像素值按之分摊，可以理解为归一化。

不出意外的话输出的图片会是这样:

![result](https://github.com/tizengyan/images/raw/master/sample1.png)

再补充几个知识点，`vec3`实现了向量的叉乘，向量叉乘的定义如下：

![vector_cross](https://github.com/tizengyan/images/raw/master/vector_cross.png)

ppm的全称是portable pixmap format，除此之外还有portable bitmap format (PBM)、portable graymap format (PGM)，也可以统称为portable anymap format (PNM)，在PPM文件中信息被这样保存：

![ppm_info](https://github.com/tizengyan/images/raw/master/ppm_info.png)

因此我们只要按这个格式规定好行列的像素数和格式，然后将像素的信息按RGB顺序一个个写入就可以生成一张图像了。

下面是主函数代码：

```c++
#include <iostream>
#include<fstream>

using namespace std;

int main() {
    int nx = 200;
    int ny = 100;
    fstream fs;
    fs.open("test.ppm", std::fstream::in | std::fstream::out | std::fstream::app);
    //cout << "P3\n" << nx << " " << ny << "\n255\n";
    fs << "P3\n" << nx << " " << ny << "\n255\n";
    for (int j = ny - 1; j >= 0; j--) {
        for (int i = 0; i < nx; i++) {
            //float r = float(i) / float(nx);
            //float g = float(j) / float(ny);
            //float b = 0.2;
            vec3 v(float(i) / float(nx), float(j) / float(ny), 0.2);
            int ir = int(255.99 * v[0]);
            int ig = int(255.99 * v[1]);
            int ib = int(255.99 * v[2]);
            //cout << ir << " " << ig << " " << ib << "\n";
            fs << ir << " " << ig << " " << ib << "\n";
        }
    }
    fs.close();
    return 0;
}
```

### Chapter 2: The vec3 class

然后是储存三维信息的`vec3`类的定义，这个类非常关键，后面几乎所有的操作都要用到这个类，此基础不牢，定地动山摇。除了用来记录向量的坐标，还可以储存像素的RGB值，因为它也是三维的：

```c++
class vec3 {
public:
    vec3() {}
    vec3(float e0, float e1, float e2) { e[0] = e0; e[1] = e1; e[2] = e2; }

    float e[3];

    inline float x() const { return e[0]; }
    inline float y() const { return e[1]; }
    inline float z() const { return e[2]; }
    inline float r() const { return e[0]; }
    inline float g() const { return e[1]; }
    inline float b() const { return e[2]; }

    inline const vec3& operator+() const { return *this; }
    inline const vec3& operator-() const { return vec3(-e[0], e[1], -e[2]); } // ?
    inline float operator[](int i) const { return e[i]; }
    inline float& operator[](int i) { return e[i]; }

    inline vec3& operator+=(const vec3 &v2);
    inline vec3& operator-=(const vec3 &v2);
    inline vec3& operator*=(const vec3 &v2);
    inline vec3& operator/=(const vec3 &v2);
    inline vec3& operator*=(const float t);
    inline vec3& operator/=(const float t);

    inline float length() const {
        return sqrt(e[0] * e[0] + e[1] * e[1] + e[2] * e[2]);
    }
    inline float squared_length() const {
        return e[0] * e[0] + e[1] * e[1] + e[2] * e[2];
    }
    inline void make_unit_vector();
};

inline istream& operator>>(istream &is, vec3 &t) {
    is >> t.e[0] >> t.e[1] >> t.e[2];
    return is;
}

inline ostream& operator<<(ostream &os, const vec3 &t) {
    os << t.e[0] << " " << t.e[1] << " " << t.e[2];
    return os;
}

inline void vec3::make_unit_vector() {
    float k = 1.0 / sqrt(e[0] * e[0] + e[1] * e[1] + e[2] * e[2]);
    e[0] *= k;
    e[1] *= k;
    e[2] *= k;
}

inline vec3 operator+(const vec3 &v1, const vec3 &v2) {
    return vec3(v1.e[0] + v2.e[0], v1.e[1] + v2.e[1], v1.e[2] + v2.e[2]);
}

inline vec3 operator-(const vec3 &v1, const vec3 &v2) {
    return vec3(v1.e[0] - v2.e[0], v1.e[1] - v2.e[1], v1.e[2] - v2.e[2]);
}

inline vec3 operator*(const vec3 &v1, const vec3 &v2) {
    return vec3(v1.e[0] * v2.e[0], v1.e[1] * v2.e[1], v1.e[2] * v2.e[2]);
}

inline vec3 operator/(const vec3 &v1, const vec3 &v2) {
    return vec3(v1.e[0] / v2.e[0], v1.e[1] / v2.e[1], v1.e[2] / v2.e[2]);
}

inline vec3 operator*(float t, const vec3 &v) {
    return vec3(t * v.e[0], t * v.e[1], t * v.e[2]);
}

inline vec3 operator*(const vec3 &v, float t) {
    return vec3(t * v.e[0], t * v.e[1], t * v.e[2]);
}

inline vec3 operator/(vec3 v, float t) {
    return vec3(v.e[0] / t, v.e[1] / t, v.e[2] / t);
}

inline float dot(const vec3 &v1, const vec3 &v2) {
    return v1.e[0] * v2.e[0] + v1.e[1] * v2.e[1] + v1.e[2] * v2.e[2];
}

inline vec3 cross(const vec3 & v1, const vec3 &v2) {
    return vec3(v1.e[1] * v2.e[2] - v1.e[2] * v2.e[1],
        v1.e[2] * v2.e[0] - v1.e[0] * v2.e[2],
        v1.e[0] * v2.e[1] - v1.e[1] * v2.e[0]);
}

inline vec3& vec3::operator+=(const vec3 &v) {
    e[0] += v.e[0];
    e[1] += v.e[1];
    e[2] += v.e[2];
    return *this;
}

inline vec3& vec3::operator*=(const vec3 &v) {
    e[0] *= v.e[0];
    e[1] *= v.e[1];
    e[2] *= v.e[2];
    return *this;
}

inline vec3& vec3::operator/=(const vec3 &v) {
    e[0] /= v.e[0];
    e[1] /= v.e[1];
    e[2] /= v.e[2];
    return *this;
}

inline vec3& vec3::operator-=(const vec3 &v) {
    e[0] -= v.e[0];
    e[1] -= v.e[1];
    e[2] -= v.e[2];
    return *this;
}

inline vec3& vec3::operator*=(const float t) {
    e[0] *= t;
    e[1] *= t;
    e[2] *= t;
    return *this;
}

inline vec3& vec3::operator/=(const float t) {
    e[0] /= t;
    e[1] /= t;
    e[2] /= t;
    return *this;
}

inline vec3 unit_vector(vec3 v) {
    return v / v.length();
}
```