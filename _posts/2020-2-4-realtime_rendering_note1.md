---
layout: post
title: "《Real-Time Rendering》读书笔记1"
categories: 图形学
tags: cg
excerpt: 主要内容为第二章的渲染管线和第三章的图形处理单元介绍
author: Tizeng
---

* content
{:toc}

本笔记根据本人阅读《Real-Time Rendering》第四版的理解纯手打，如果有外用内容会有标注。
第一章约定了一些数学变量、公式等的表达方式和写法，避免后面阅读时引起歧义。

## Chapter 2: The Graphics Rendering Pipeline

第二章介绍图形渲染管线，这是实时渲染的核心部分，管线的主要任务是从给定的虚拟摄像机、三维物体、光源灯条件下生成（渲染）出一幅二维的图像。这个过程很好理解，具体见前面有关光追的读书笔记中有记录和实现。

### 2.2应用阶段（Application Stage）

### 2.3几何处理阶段（Geometry Processing）

#### 顶点着色

#### 可选顶点处理

在顶点处理（vertex processoing）结束后GPU中有一些可选阶段可以执行，按顺序分别是tesselation、geometry shader、stream output。

tesselation：密铺、镶嵌，指在一个较大的面中填充较小的面且不留空隙，它有很多种套路和算法，在用三角形生成曲面时我们为了在质量（quality）和效率（performance）上取得平衡，可能会使用到这个技术。

#### 裁剪（Clipping）

完全在摄像机视野（view volume）外的图元（primitives）会被丢弃，完全在视野内的图元会原封不动的pass到下一个阶段，那些一部分在外一部分在内的图元就需要被裁剪，只留下视野内的部分。最终，透视分割（perspective division）的执行让（待定）

几何阶段的最后一步就是将坐标空间转化成窗口坐标。

#### 屏幕映射（Screen Mapping）

图元进入这个阶段后，坐标仍是三维的，每个图元的xy坐标会转化成屏幕坐标（screen coordinates），带z坐标的屏幕坐标也被称为窗口坐标（window coordinates）。转化中通常会进行拉伸（scaling）操作，z坐标会根据不同标准映射到不同的范围。

考虑一排水平摆放的单排像素，最左边像素的左边缘坐标是0.0，那么该像素的中心是0.5，因此范围[0, 9]的像素可以覆盖[0.0, 10.0)的区域。OpenGL和DX定义的坐标系方向不同，OpenGL中左下角是最小值（lowest-valued）元素，而DX是左上，都是合理的。

### 2.4光栅化（Rasterization）

光栅化阶段的目的是找到图元内部的所有像素，分为两个子阶段，图元装配（primitive assembly）和三角形遍历（triangle traversal），第一个子阶段又叫三角形安装（triangle setup）。

### 2.5像素处理阶段（Pixel Processing）



## Chapter 3
