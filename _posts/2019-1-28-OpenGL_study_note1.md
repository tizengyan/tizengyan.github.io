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

由于“[Learn OpenGL](learnopengl.com/)”网站上对于OpenGL的教程已经十分细致，具体的操作流程和函数解释就不过多赘述，这里只记录比较重要的信息和跑出来的代码。

## OpenGL的配置

OpenGL本身是一套标准/规范，为了能在多种平台运行，它并不规定如何在我们电脑上的窗口显示，因此需要用到一些第三方库来显示窗口，这里我们用GLFW来显示窗口，另外还需要GLAD来管理OpenGL的函数地址，具体的配置流程请看[这里](https://learnopengl-cn.github.io/01%20Getting%20started/02%20Creating%20a%20window/)。

## 创建一个窗口

代码含义见注释，或者直接参阅[这里](https://learnopengl-cn.github.io/01%20Getting%20started/03%20Hello%20Window/)。

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

void showWindowTest(GLFWwindow *window) {
	while (!glfwWindowShouldClose(window)) { // 检查是否需要退出
		processInput(window); // 处理输入
		glClearColor(0.2f, 0.3f, 0.3f, 1.0f); // 设置清屏状态
		glClear(GL_COLOR_BUFFER_BIT); // 使用清屏状态进行清屏

		glfwSwapBuffers(window); // 绘制像素，swap表示使用双缓冲double buffer
								 // 这样可以对用户隐藏绘制像素的过程，直接交换上一次绘制好的图像，从而消除屏幕的闪烁
		glfwPollEvents(); // 检查触发事件
	}

	glfwTerminate(); // 释放资源
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
	if (!gladLoadGLLoader((GLADloadproc)glfwGetProcAddress)) { // 初始化glad
		cerr << "failed to init glad" << endl;
		return -1;
	}

	// 注册回调函数，每次窗口大小改变时调用
	glfwSetFramebufferSizeCallback(window, framebuffer_size_callback);

	showWindowTest(window);
	//drawTriangle(window);

	return 0;
}
```

![效果图](https://github.com/tizengyan/images/raw/master/OpenGL_create_window.png)

## [画出三角形](https://learnopengl-cn.github.io/01%20Getting%20started/04%20Hello%20Triangle/)

三角形是图形学中的基本单位，几乎所有的图像都由三角形构成。

先记住以下三个概念：

* 顶点数组对象：Vertex Array Object，VAO

* 顶点缓冲对象：Vertex Buffer Object，VBO

* 索引缓冲对象：Element Buffer Object，EBO 或 Index Buffer Object，IBO

然后复习一下渲染管线和着色器（shader）：

![效果图](https://github.com/tizengyan/images/raw/master/OpenGL_pipeline.png)

流程依次为：顶点着色器 -> 图元装配（Primitive Assembly）-> 几何着色器 -> 光栅化 -> 片段着色器 -> 测试与混合

为了画出一个三角形，我们首先需要三个点在空间中的坐标，注意只有坐标值在**标准化设备坐标**(Normalized Device Coordinates)范围内的坐标才会最终呈现在屏幕上，在[-1, 1]范围以外的坐标都不会显示。
用数组储存顶点的坐标如下：

```c++
float vertices[] = {
    -0.5f, -0.5f, 0.0f,
     0.5f, -0.5f, 0.0f,
     0.0f,  0.5f, 0.0f,
};
```

标准化设备坐标之后会变成屏幕空间坐标(Screen-space Coordinates)，然后又会变换为片段输入片段着色器中。VBO负责管理储存在GPU内存中的顶点信息。

我们需要用到着色器语言GLSL(OpenGL Shading Language)来编写顶点着色器，它的示例语法如下，为了能编译它，我们将其储存在一个字符串中：

```c++
const char* vertexShaderSource =
"#version 460 core\n"
"layout(location = 0) in vec3 aPos;\n"
"void main(){\n"
"    gl_Position = vec4(aPos.x, aPos.y, aPos.z, 1.0);\n"
"}\0";

const char* fragmentShaderSource =
"#version 460 core\n"
"out vec4 FragColor;\n"
"void main(){\n"
"    FragColor = vec4(1.0f, 0.5f, 0.2f, 1.0f);\n"
"}\0";
```

片段着色器的任务是计算最后像素应该以什么颜色输出，这里我们为了方便观察调试把颜色写死成橘色。两个着色器编译完成后，下一步是将它们链接到一个用来渲染的着色器程序（Shader Program）中，这样我们才能使用它们，链接完成后，之前的着色器对象就可以删除了。然后我们要告诉OpenGL如何解析顶点数据，并启用顶点属性，才能开始绘制图像，如果每次绘制都要进行一遍这样的操作，属实麻烦，这时就需要VAO来储存状态配置，它可以像VBO那样被绑定，不同顶点数据和配置绑定不同的VAO，绘制完之后记得解绑VAO供之后使用。注意VAO的绑定需要在VBO的绑定之前，否则会看不到三角形。

下面是完整代码和输出结果：

```c++
void drawTriangle(GLFWwindow* window) {
	const char* vertexShaderSource = // 顶点着色器源码
		"#version 460 core\n"
		"layout (location = 0) in vec3 aPos;\n"
		"void main() {\n"
		"    gl_Position = vec4(aPos.x, aPos.y, aPos.z, 1.0);\n"
		"}\0";
	const char* fragmentShaderSource = // 片段着色器源码
		"#version 460 core\n"
		"out vec4 FragColor;\n"
		"void main() {\n"
		"    FragColor = vec4(1.0f, 0.5f, 0.2f, 1.0f);\n"
		"}\0";

	unsigned int vertexShader;
	vertexShader = glCreateShader(GL_VERTEX_SHADER); // 创建顶点着色器
	glShaderSource(vertexShader, 1, &vertexShaderSource, NULL); // 编译顶点着色器
	glCompileShader(vertexShader);
	int success;
	char infoLog[512];
	glGetShaderiv(vertexShader, GL_COMPILE_STATUS, &success); // 检查编译是否成功
	if (!success) {
		glGetShaderInfoLog(vertexShader, 512, NULL, infoLog); // 如果不成功，获取错误信息
		cout << "Failed to compile vertex shader source code: \n" << infoLog << endl;
	}

	unsigned int fragmentShader;
	fragmentShader = glCreateShader(GL_FRAGMENT_SHADER);
	glShaderSource(fragmentShader, 1, &fragmentShaderSource, NULL); // 编译片段着色器
	glCompileShader(fragmentShader);
	glGetShaderiv(fragmentShader, GL_COMPILE_STATUS, &success); // 检查编译是否成功
	if (!success) {
		glGetShaderInfoLog(fragmentShader, 512, NULL, infoLog); // 如果不成功，获取错误信息
		cout << "Failed to compile fragment shader source code: \n" << infoLog << endl;
	}
	unsigned int shaderProgram; // 创建着色器程序
	shaderProgram = glCreateProgram(); // 返回对象的id引用
	glAttachShader(shaderProgram, vertexShader); // 添加着色器到着色器程序
	glAttachShader(shaderProgram, fragmentShader);
	glLinkProgram(shaderProgram); // 链接绑定的着色器
	glGetProgramiv(shaderProgram, GL_LINK_STATUS, &success);
	if (!success) {
		glGetProgramInfoLog(shaderProgram, 512, NULL, infoLog);
		cout << "Failed to link shaders: \n" << infoLog << endl;
	}
	glDeleteShader(vertexShader); // 删除着色器对象
	glDeleteShader(fragmentShader);

	float vertices[] = {
		-0.5f, -0.5f, 0.0f,
		 0.5f, -0.5f, 0.0f,
		 0.0f,  0.5f, 0.0f,
	};

	unsigned int VBO, VAO;
	glGenBuffers(1, &VBO); // 生成VBO缓冲对象
	glGenVertexArrays(1, &VAO); // 创建顶点数组对象
	glBindVertexArray(VAO); // 绑定顶点数组对象

	glBindBuffer(GL_ARRAY_BUFFER, VBO); // 绑定为GL_ARRAY_BUFFER缓冲类型（顶点缓冲）
										// 完成绑定后，任何`GL_ARRAY_BUFFER`的缓冲调用都会用来配置当前绑定的缓冲
	glBufferData(GL_ARRAY_BUFFER, sizeof(vertices), vertices, GL_STATIC_DRAW); // 将顶点数据复制到缓冲内存

	// 解析顶点数据：
	// 顶点着色器源码将location设置为了0；vec3；参数为浮点型；normalize；步长（这里数据紧密排列）；缓冲起始位置offset
	glVertexAttribPointer(0, 3, GL_FLOAT, GL_FALSE, 3 * sizeof(float), (void*)0);
	glEnableVertexAttribArray(0); // 顶点属性起始位置，启用顶点属性（默认为禁用）

	glPolygonMode(GL_FRONT_AND_BACK, GL_LINE); // 只画线

	while (!glfwWindowShouldClose(window)) {
		processInput(window);
		glClearColor(0.2f, 0.3f, 0.3f, 1);
		glClear(GL_COLOR_BUFFER_BIT);

		glUseProgram(shaderProgram); // 激活程序对象，可以放循环外面
		//glBindVertexArray(VAO); // 这里我们只有一个VAO不需要每次绑定和解绑，但还是规范一点
		glDrawArrays(GL_TRIANGLES, 0, 3);
		//glDrawElements(GL_TRIANGLES, 6, GL_UNSIGNED_INT, 0); // 根据索引绘制，最后一个参数为缓冲的offset
		//glBindVertexArray(0); // 解绑

		glfwSwapBuffers(window);
		glfwPollEvents();
	}
	glDeleteBuffers(1, &VBO);
	glDeleteVertexArrays(1, &VAO);

	glfwTerminate();
}
```

![效果图](https://github.com/tizengyan/images/raw/master/OpenGL_create_triangle.png)

如果我们要画一个矩形，可以用画两次三角形来实现，比如将输入的顶点坐标改成：

```c++
float vertices[] = {
    // 第一个三角形
     0.5f,  0.5f, 0.0f,   // 右上角
     0.5f, -0.5f, 0.0f,   // 右下角
    -0.5f,  0.5f, 0.0f,   // 左上角
    // 第二个三角形
     0.5f, -0.5f, 0.0f,  // 右下角
    -0.5f, -0.5f, 0.0f,  // 左下角
    -0.5f,  0.5f, 0.0f   // 左上角
};
```

这样的问题是重复声明了两个点，造成额外开销，我们实际上只需要四个点，这时就轮到EBO出场了，它记录的是索引，也就是OpenGL绘制图像的顺序，这样在`vertices`中我们就只需要声明四个点和一个索引数组：

```c++
float vertices[] = {
	 0.5f,  0.5f, 0.0f,  // 右上角
	 0.5f, -0.5f, 0.0f,  // 右下角
	-0.5f, -0.5f, 0.0f,  // 左下角
	-0.5f,  0.5f, 0.0f,  // 左上角
};

unsigned int indices[] = {
	0, 1, 3,  // 第一个三角形
	1, 2, 3,  // 第二个三角形
};
```

EBO的绑定要在VBO完成绑定等操作之后，最后将绘图的函数从`glDrawArray`换成`glDrawElements`即可。

```c++
unsigned int EBO;
glGenBuffers(1, &EBO); // 创建索引缓冲对象
glBindBuffer(GL_ELEMENT_ARRAY_BUFFER, EBO); // 绑定缓冲
glBufferData(GL_ELEMENT_ARRAY_BUFFER, sizeof(indices), indices, GL_STATIC_DRAW); // 把索引复制到缓冲
```

![效果图](https://github.com/tizengyan/images/raw/master/OpenGL_create_rectangle.png)

### 思考题



## Debug日志

### (1) LINK1104找不到glfw3.lib

原因是编译好的库文件glfw3.lib的目录没有被正确包含进vs的项目中，Library的目录要在project属性中的Library Directories包含而非Include Directories中包含。

### (2) unresolved external symbol _glfwinit referenced in function _main

编译生成的glfw3.lib库文件是64bit而vs项目运行在32bit环境下，可以将vs设置为x64再运行，或从GLFW官网下载32bit的二进制文件。
