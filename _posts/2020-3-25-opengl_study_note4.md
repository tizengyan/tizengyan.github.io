---
layout: post
title:  "OpenGL学习笔记4——空间变换"
date:   2020-04-09
categories: 图形学
tags: OpenGL
excerpt: 实现简单的2d与3d变换
author: Tizeng
---

* content
{:toc}

<head>
    <script src="https://cdn.mathjax.org/mathjax/latest/MathJax.js?config=TeX-AMS-MML_HTMLorMML" type="text/javascript"></script>
    <script type="text/x-mathjax-config">
        MathJax.Hub.Config({
            tex2jax: {
            skipTags: ['script', 'noscript', 'style', 'textarea', 'pre'],
            inlineMath: [['$','$']]
            }
        });
    </script>
</head>

## 2D旋转

这里直接用前面笔记中加载纹理的代码，vertex shader中使用`mat4`表示旋转矩阵来对顶点的位置进行变换：

```c++
#version 460 core
layout (location = 0) in vec3 aPos;
layout (location = 1) in vec3 aColor;
layout (location = 2) in vec2 aTexCoord;

out vec3 myColor;
out vec2 myTexCoord;
uniform mat4 transform;

void main() {
    gl_Position = transform * vec4(aPos, 1.0f);
    myColor = aColor;
    myTexCoord = vec2(aTexCoord.x, 1.0f - aTexCoord.y);
}
```

然后在循环中根据时间来设置旋转的角度，注意这里每次都在循环内部创建变换矩阵`trans`，而不是外部创建好之后重复使用，否则每次旋转的角度会一直叠加导致旋转的速度飞快不受控制。

```c++
// ...
while(...) {
    //...
    glm::mat4 trans;
	trans = glm::translate(trans, glm::vec3(-0.5, 0.5, 0.0)); // 位移
    trans = glm::scale(trans, glm::vec3(0.5, 0.5, 0.5)); // 缩放
    trans = glm::rotate(trans, float(glfwGetTime()), glm::vec3(0.0, 0.0, 1.0)); // 旋转
    glUniformMatrix4fv(transformLoc, 1, GL_FALSE, glm::value_ptr(trans));
    //...
}
//...
```

尽管代码中是先位移，再缩放，最后旋转，但运行时实际上是反过来的，就像多个变换矩阵相乘时，发生变换的顺序是最先和目标向量相乘的矩阵，也就是从右往左，而不是习惯上的从左往右。

## 坐标变换

顶点坐标在被变换到屏幕坐标时要经历多个坐标系统，它们是：局部空间 -> 世界空间 -> 观察空间 -> 裁剪空间 -> 屏幕空间。每个坐标系统有它处理某些操作的方便之处，变换时通过特定的矩阵来变换坐标系，模型矩阵、观察矩阵、投影矩阵负责前三步的变换，最后通过视口变换（viewport transform），坐标被映射到屏幕空间中，然后被送到光栅器，将其转化为片段。

在绘制循环中加入下面的代码，分别定义模型矩阵、观察矩阵和投影矩阵，就可以得到一个沿着向量(0.5f, 1.0f, 0.0f)转动的箱子：

```c++
// ...
glm::mat4 model, view, projection;
model = glm::rotate(model, (float)glfwGetTime() * glm::radians(50.0f), glm::vec3(0.5f, 1.0f, 0.0f));
view = glm::translate(view, glm::vec3(0.0f, 0.0f, -5.0f));
// 创建投影矩阵：视野field of view、宽高比、近距离、远距离
projection = glm::perspective(glm::radians(45.0f), screenWidth / screenHeight, 0.1f, 100.0f);
myShader.setMat4("model", model);
myShader.setMat4("view", view);
myShader.setMat4("projection", projection);
// ...

void setMat4(const string& name, glm::mat4 mat) const {
    glUniformMatrix4fv(glGetUniformLocation(ID, name.c_str()), 1, GL_FALSE, glm::value_ptr(mat));
}
```

新的顶点shader，注意矩阵与顶点坐标相乘的顺序：

```c++
#version 460 core
layout (location = 0) in vec3 aPos;
layout (location = 1) in vec2 aTexCoord;

out vec2 myTexCoord;
uniform mat4 model;
uniform mat4 view;
uniform mat4 projection;

void main() {
    gl_Position = projection * view * model * vec4(aPos, 1.0f);
    myTexCoord = vec2(aTexCoord.x, aTexCoord.y);
}
```

下面一个一个来了解每个矩阵的原理。

### 投影矩阵

这部分的内容主要参考[songho的网站](http://www.songho.ca/opengl/gl_projectionmatrix.html)。为了让画面有透视的效果（近大远小），需要用到前面定义的齐次坐标，即除了xyz之外的第四个坐标w，在观察坐标乘以投影矩阵变为裁剪坐标（clip coordinates）后，它仍是齐次坐标，最后将xyz三个分量分别除以w，称为透视除法，就得到了标准化设备坐标（normalized device coordinates），准备映射到屏幕上了。

其中推导坐标的关系有点绕，做一下简单的记录：首先观察空间的齐次坐标是四维的，而与投影矩阵相乘后也应该得到一个四维坐标，因此投影矩阵一定是一个4x4的矩阵，我们最后要得到的是一组裁剪空间的坐标$(x_c, y_c, z_c, w_c)$，用其可以变幻出标准化设备坐标$(\frac{x_c}{w_c}, \frac{y_c}{w_c}, \frac{z_c}{w_c})$。在观察空间有一个坐标记为$(x_e, y_e, z_e)$（e for eye），我们把它通过相似三角形的关系投影到仅屏幕near plane上，记为$(x_p, y_p, z_p)$（p for plane），这样就用观察坐标表示了p坐标，而x与y的p坐标总是需要用e坐标除以$-z_e$来得到，因此不妨假定$w_c=-z_e$，由此得到了矩阵的最后一行。假设近屏幕X轴的范围是[l, r]，Y轴的范围是[b, t]，以观察点为原点，到远屏幕的范围是[-n, -f]，那么我们为了将其上的坐标等比例映射到[-1, 1]的范围内得到标准化设备坐标，可以用简单的线性关系对x和y进行映射，这样就可以得到矩阵的前两行。现在只剩第三行，对于z来说，其实只要落在近屏幕上它就总是-n，但这样会丢失坐标的深度信息，这显然是不行的，因此我们要找出一个对$z_c$的唯一映射关系。很明显它应该与xy方向的值无关，因此第三行前两个元素是0，后两个元素设为两个未知数，对于$z_e$和$z_n$来说，要把它们从[-n, -f]映射到[-1, 1]（注意这里从右手坐标系变成了左手），相当于知道了直线上的两个点，就能求解两个未知数了，如此一来就能得出透视投影矩阵，将其与观察空间中的坐标相乘，便能得到裁剪坐标。需要注意的是$z_e$和$z_n$并非成线性关系，如果[-n, -f]差距太大，会造成即使$z_e$变化很大，$z_n$也几乎不变的问题，称为深度精确问题（z-fighting）。

说完了透视投影，接下来是正交投影，正交投影三个维度都是线性映射，而且不存在前面的相似三角形的情况，因此$w_c=w_e$。此时的$z_e$和$z_n$也是线性关系，依旧要注意坐标系从右手变成了左手，-n对应的是-1。

### 观察矩阵

观察空间是按摄像机的位置为原点建立的坐标系，同样是右手坐标系，但z轴正方向与摄像机的视线方向相反，该方向的向量可以由摄像机坐标减去目标坐标得到，然后可以根据世界坐标的上向量与z轴向量叉乘，得到右轴也就是x轴向量，再用z轴叉乘x轴得到y轴向量。因此我们需要三个向量来获得观察矩阵：摄像机坐标、目标**坐标**、上向量（都以世界空间为坐标系）。

#### 变换原理

我们最终的目的是构造出一个矩阵，让世界坐标中的向量与之相乘之后可以变换到以摄像机为原点的坐标系中。对于一个n维方阵来说，如果将其每行看作是新坐标系的**基向量**，那么将原始坐标系中的向量与之相乘就能将它转换为新坐标系中的向量，由于零向量乘以任何矩阵依然是零向量，因此变换后的坐标系的原点与原坐标系的原点一致，即这种变换不包括平移，包含平移的变换称为仿射变换，仿射变换需要将矩阵添加一维，然后将三维向量也添加一个w作为齐次坐标，这样我们可以通过控制w的值来决定向量与变换矩阵相乘后是否被位移所影响。

#### 使用lookAt矩阵

有了lookAt矩阵，就可以让摄像机做各种各样的变换了，移动部分，加上想要移动方向的向量就可以，根据所需的距离控制模长即可。若想达到在移动时摄像机视线跟随平移，只需要将目标坐标加上摄像机的实时坐标即可，至于原因其实很简单，设想摄像机在原点，看向目标所在的方向就是目标坐标代表的向量方向，当摄像机移动时，其坐标与目标坐标所代表的向量和为它们所组成平行四边形的中间线的向量，此时计算出的摄像机方向为平行四边形的另一边，与原始目标坐标向量平行。解释起来有点绕，画个图就清楚了，关键在于我们期望的视线方向是与目标坐标向量**平行**的方向，即新的位置看向新的目标坐标的方向，而不是新位置看向老目标坐标的方向，因为我们最开始定义目标坐标时，是根据摄像机在原点时的视角决定的。

现在要实现一个通过鼠标的移动来改变摄像头角度的效果，就像FPS的操作那样，且可以朝着摄像机看着的方向移动。这种情况只需要考虑欧拉角中的偏航角和俯仰角，注意偏航角是从x轴的正方向为起点，朝z轴正方向旋转为正方向！因此当摄像机初始视线看向z轴负方向时，应初始化yaw为-90度。在每帧捕获键盘输入后，将摄像机坐标加上一定比例的目标坐标，以这样的方式实现移动，就可以在目标位置改变时，继续朝那个方向移动了。
