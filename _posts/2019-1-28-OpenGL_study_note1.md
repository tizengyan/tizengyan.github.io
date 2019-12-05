---
layout: post
title:  "OpenGL学习笔记1"
date:   2019-02-08 20:34:54
categories: 图形学
tags: OpenGL
excerpt: OpenGL配置、创建窗口、画出直线和三角形
author: Tizeng
---

* content
{:toc}

由于“[learn OpenGL](learnopengl.com/)”网站上对于OpenGL的教程已经十分细致，具体的操作流程和函数解释就不过多赘述，这里只记录比较重要的信息和跑出来的代码。

## OpenGL的配置

OpenGL本身是一套标准/规范，为了能在多种平台运行，它并不规定如何在我们电脑上的窗口显示，因此需要用到一些第三方库来显示窗口，这里我们用GLFW来显示窗口，另外还需要GLAD来查询OpenGL的函数地址，具体的配置流程请看[这里](https://learnopengl-cn.github.io/01%20Getting%20started/02%20Creating%20a%20window/)。

## 创建一个窗口

具体的代码解释在[这里](https://learnopengl-cn.github.io/01%20Getting%20started/03%20Hello%20Window/)。

```c++
#include <glad/glad.h>
#include <GLFW/glfw3.h>
#include <iostream>
using namespace std;

void framebuffer_size_callback(GLFWwindow* window, int width, int height) {
	glViewport(0, 0, width, height);
}

void processInput(GLFWwindow* window) {
	if (glfwGetKey(window, GLFW_KEY_ESCAPE) == GLFW_PRESS) { // 检查是否escape被按下
		glfwSetWindowShouldClose(window, true); // 如果按下则告诉glfw关闭窗口
	}
}

int main() {
	glfwInit();
	glfwWindowHint(GLFW_CONTEXT_VERSION_MAJOR, 4);
	glfwWindowHint(GLFW_CONTEXT_VERSION_MINOR, 6);
	glfwWindowHint(GLFW_OPENGL_PROFILE, GLFW_OPENGL_CORE_PROFILE);
	GLFWwindow* window = glfwCreateWindow(800, 600, "test", NULL, NULL);
	if (window == NULL) {
		cout << "window create failed" << endl;
		glfwTerminate();
		return -1;
	}
	glfwMakeContextCurrent(window); // 通知glfw将window的context设置为当前线程的主context
	if (!gladLoadGLLoader((GLADloadproc)glfwGetProcAddress)) {
		cerr << "failed to init glad" << endl;
		return -1;
	}

	// 注册回调函数，每次窗口大小改变时调用
	glfwSetFramebufferSizeCallback(window, framebuffer_size_callback);

	while (!glfwWindowShouldClose(window)) { // 检查是否需要退出
		processInput(window); // 处理输入
		glClearColor(0.2f, 0.3f, 0.3f, 1.0f); // 设置清屏状态
		glClear(GL_COLOR_BUFFER_BIT); // 清屏

		glfwSwapBuffers(window); // 绘制像素，swap表示使用双缓冲double buffer
								 //这样可以对用户隐藏绘制像素的过程，直接交换上一次绘制好的图像，从而消除屏幕的闪烁
		glfwPollEvents(); // 检查触发事件
	}

	glfwTerminate(); // 释放资源

	return 0;
}
```

![效果图](https://github.com/tizengyan/images/raw/master/OpenGL_create_window.png)

## 画出三角形

三角形是图形学中的基本单位，几乎所有的图像都由三角形构成。

先记住以下三个概念：

* 顶点数组对象：Vertex Array Object，VAO

* 顶点缓冲对象：Vertex Buffer Object，VBO

* 索引缓冲对象：Element Buffer Object，EBO 或 Index Buffer Object，IBO

然后复习一下渲染管线和着色器（shader）：

![效果图](https://github.com/tizengyan/images/raw/master/OpenGL_pipeline.png)

流程依次为：顶点着色器 -> 图元装配（Primitive Assembly）-> 几何着色器 -> 光栅化 -> 片段着色器 -> 测试与混合

为了画出一个三角形，我们首先需要三个点在空间中的坐标，注意只有坐标值在标准化设备坐标(Normalized Device Coordinates)范围内的坐标才会最终呈现在屏幕上，在[-1, 1]范围以外的坐标都不会显示。
用数组储存顶点的坐标如下：

```c++
float vertices[] = {
    -0.5f, -0.5f, 0.0f,
    0.5f, -0.5f, 0.0f,
    0.0f,  0.5f, 0.0f
};
```

我们需要用到着色器语言GLSL(OpenGL Shading Language)来编写顶点着色器，它的示例语法如下，为了能编译它，我们将其储存在一个字符串中：

```c++
const char* vertexShaderSource =
"#version 460 core\n"
"layout(location = 0) in vec3 aPos;\n"
"void main(){\n"
"   gl_Position = vec4(aPos.x, aPos.y, aPos.z, 1.0);\n"
"}\0";

const char* fragmentShaderSource =
"#version 460 core\n"
"out vec4 FragColor;\n"
"void main(){\n"
"   FragColor = vec4(1.0f, 0.5f, 0.2f, 1.0f);\n"
"}\0";
```

### 索引缓冲对象



## Debug日志

### (1) LINK1104找不到glfw3.lib

原因是编译好的库文件glfw3.lib的目录没有被正确包含进vs的项目中，Library的目录要在project属性中的Library Directories包含而非Include Directories中包含。

### (2) unresolved external symbol _glfwinit referenced in function _main

编译生成的glfw3.lib库文件是64bit而vs项目运行在32bit环境下，可以将vs设置为x64再运行，或从GLFW官网下载32bit的二进制文件。
