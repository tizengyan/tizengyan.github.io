---
layout: post
title:  "OpenGL学习笔记2"
date:   2020-01-14
categories: 图形学
tags: OpenGL
excerpt: shader，着色器语言GLSL
author: Tizeng
---

* content
{:toc}

上一篇笔记的创建时间正好是去年的这个时候，相隔不过几天，拖到现在才把它完成开始写第二篇，实属不应该。

## GLSL

翻译为GL着色器语言，是一种语法和C很像的语言，主要是规定各个shader如何工作。着色器是运行在GPU上的小程序，在外界看来就是将输入转化为输出，它们是相互独立的，只能通过输入和输出沟通，比如在顶点着色器的源码中规定一个输出（out）颜色变量，然后在片段着色器中声明一个同样名称的输入（in）颜色变量，这样OpenGL就会把顶点着色器中的颜色传入片段着色器中了。为了便于理解直接看下面的代码，直接在上一篇绘制三角形的代码基础上做修改：

```c++
const char* vertexShaderSource = // 顶点着色器源码
    "#version 460 core\n"
    "layout (location = 0) in vec3 aPos;\n" // 通过location指定输入
    "layout (location = 1) in vec3 aColor;\n"
    "out vec3 vertexColor;\n"
    "void main() {\n"
    "    gl_Position = vec4(aPos, 1.0);\n"
    "	 vertexColor = aColor;\n"
    "}\0";
const char* fragmentShaderSource = // 片段着色器源码
    "#version 460 core\n"
    "in vec3 vertexColor; \n"
    "out vec4 FragColor;\n"
    //"uniform vec4 myColor;\n"
    "void main() {\n"
    //"    FragColor = vec4(1.0f, 0.5f, 0.2f, 1.0f);\n"
    //"	 FragColor = myColor;\n"
    "    FragColor = vec4(vertexColor, 1.0);\n"
    "}\0";

float vertices1[] = {
    // 位置             颜色
    -0.9f, -0.5f, 0.0f, 1.0, 0.0, 0.0, // left 
    -0.0f, -0.5f, 0.0f, 0.0, 1.0, 0.0, // right
    -0.45f, 0.5f, 0.0f, 0.0, 0.0, 1.0, // top 
};

// ...

// 位置属性
// location为0，分量为3，分量类型为浮点型，不标准化，到下一个顶点属性的步长是6，偏移量为0
glVertexAttribPointer(0, 3, GL_FLOAT, GL_FALSE, 6 * sizeof(float), (void*)0); 
glEnableVertexAttribArray(0);
// 颜色属性
// location为1，分量为3，浮点，不标准化，到下一个颜色属性步长为6，结构为三个坐标跟着三个颜色以此往复，因此到颜色的步长为3
glVertexAttribPointer(1, 3, GL_FLOAT, GL_FALSE, 6 * sizeof(float), (void*)(3 * sizeof(float))); 
glEnableVertexAttribArray(1);

// ...
```

更改代码后新的效果图如下：

![效果图](https://github.com/tizengyan/images/raw/master/opengl_colored_triangle.png)

我们在顶点数据中加入了颜色的数据，和坐标一样三个分量，因此现在一个顶点数据现在有六个值，三个坐标值三个颜色值，这样一来距下一个顶点数据的步长（stride）就是6，由于位置数据在前，颜色数据在后，因此位置属性的偏移量（offset）为0，颜色属性的偏移量为3。在顶点着色器源码中的location就是标记位置和颜色的信息。

注意最终得到的三角形中并不是只有单纯的rgb三个颜色，而是在中间发生了渐变，越靠近顶点越接近该顶点数据中的颜色，这是因为在片段着色器中进行了插值（interpolate）的效果，一般来说是线性插值，以下是维基百科对插值的描述：

    插值（interpolation）是一种通过已知的、离散的数据点，在范围内推求新数据点的过程或方法

这里已知的数据点就是顶点的颜色，而顶点中间像素的颜色是未知的，我们就用这已知的三个颜色对其他的像素进行插值得出对应的颜色。

## shader类

每次处理和编译处理shader源码比较麻烦，这个过程可以交给一个类来完成，然后把shader的源码放进单独的文件中编写，由这个类来读取，这样可以增加我们的开发效率，下面是代码：

```c++
#pragma once
#ifndef SHADER_H
#define SHADER_H

#include <glad/glad.h>
#include <fstream>
#include <iostream>
#include <sstream>

using namespace std;

class MyShader {
private:
	unsigned int ID;

public:
	MyShader(const GLchar* vertexPath, const GLchar* fragmentPath) {
		string vertexCode, fragmentCode; // 源码缓存
		ifstream vShaderFile, fShaderFile; // 源码文件流
		vShaderFile.exceptions(ifstream::failbit | ifstream::badbit); // 注册异常
		fShaderFile.exceptions(ifstream::failbit | ifstream::badbit);
		try {
			vShaderFile.open(vertexPath); // 读取源码文件
			fShaderFile.open(fragmentPath); 
			stringstream vShaderStream, fShaderStream; // 数据流
			vShaderStream << vShaderFile.rdbuf(); // 将文件缓冲给到我们的数据流
			fShaderStream << fShaderFile.rdbuf();
			vShaderFile.close();
			fShaderFile.close();
			vertexCode = vShaderStream.str(); // 从数据流中读取字符串
			fragmentCode = fShaderStream.str();
		}
		catch (ifstream::failure e) {
			cout << "Source file not successfully read. " << endl;
		}

		const char* vShaderCode = vertexCode.c_str();
		const char* fShaderCode = fragmentCode.c_str();
		unsigned int vShader, fShader;
		int success;
		char infoLog[512];
		vShader = glCreateShader(GL_VERTEX_SHADER); // 创建
		glShaderSource(vShader, 1, &vShaderCode, NULL); // 加载
		glCompileShader(vShader); // 编译
		glGetShaderiv(vShader, GL_COMPILE_STATUS, &success);
		if (!success) {
			glGetShaderInfoLog(vShader, 512, NULL, infoLog);
			cout << "Vertex shader compilation failed: " << infoLog << endl;
		}
		fShader = glCreateShader(GL_FRAGMENT_SHADER);
		glShaderSource(fShader, 1, &fShaderCode, NULL);
		glCompileShader(fShader);
		glGetShaderiv(fShader, GL_COMPILE_STATUS, &success);
		if (!success) {
			glGetShaderInfoLog(fShader, 512, NULL, infoLog);
			cout << "Fragment shader compilation failed: " << infoLog << endl;
		}

		// 创建着色器程序
		ID = glCreateProgram();
		glAttachShader(ID, vShader); // 添加顶点着色器
		glAttachShader(ID, fShader); // 添加片段着色器
		glLinkProgram(ID); // 链接
		glGetProgramiv(ID, GL_LINK_STATUS, &success);
		if (!success) {
			glGetProgramInfoLog(ID, 512, NULL, infoLog);
			cout << "Failed to link shader program: " << infoLog << endl;
		}

		glDeleteShader(vShader);
		glDeleteShader(fShader);
	}

	void use() {
		glUseProgram(ID);
	}

	// uniform functions, if needed
	void setInt(const string& name, int val) const {
		glUniform1i(glGetUniformLocation(ID, name.c_str()), val);
	}
	void setBool(const string& name, bool val) const {
		glUniform1i(glGetUniformLocation(ID, name.c_str()), val);
	}
	void setFloat(const string& name, float val) const {
		glUniform1f(glGetUniformLocation(ID, name.c_str()), val);
	}
};

#endif
```

剩下的就是使用这个类，注意它只是帮我们读取了shader的源码并链接到shader程序上供我们使用，要绘制的坐标点、VAO和VBO这些还是要额外处理，这里正好复习一下前面VBO和VAO的操作顺序，先创建VBO和VAO，**记住先绑定VAO**，再绑定VBO，读取顶点数据，告诉opengl如何处理这些数据（glVertexAttribPointer），根据location的值启用顶点属性，然后是绘制循环，在循环中将激活shader程序这一步改成我们的shader类来处理就行了，修改过的绘制函数如下：

```c++
void drawTriangle2(GLFWwindow* window) {
	MyShader myShader("D:\\Projects\\OpenGLTest\\OpenGLTest\\vertexShaderSource", "D:\\Projects\\OpenGLTest\\OpenGLTest\\fragmentShaderSource");
	float vertices[] = {
		// first triangle
		-0.5f, -0.5f, 0.0f, 1.0f, 0.0f, 0.0f, // left 
		 0.5f, -0.5f, 0.0f, 0.0f, 1.0f, 0.0f, // right
		 0.0f,  0.5f, 0.0f, 0.0f, 0.0f, 1.0f, // top 
	};
	unsigned int VBO, VAO;
	glGenBuffers(1, &VBO);
	glGenVertexArrays(1, &VAO);
	glBindVertexArray(VAO); // 先绑定VAO
	glBindBuffer(GL_ARRAY_BUFFER, VBO); // 再绑定VBO
	glBufferData(GL_ARRAY_BUFFER, sizeof(vertices), vertices, GL_STATIC_DRAW); // 读取顶点数据
	glVertexAttribPointer(0, 3, GL_FLOAT, GL_FALSE, 6 * sizeof(float), (void*)0);
	glEnableVertexAttribArray(0);
	glVertexAttribPointer(1, 3, GL_FLOAT, GL_FALSE, 6 * sizeof(float), (void*)(3 * sizeof(float)));
	glEnableVertexAttribArray(1);

	while (!glfwWindowShouldClose(window)) {
		processInput(window);
		glClearColor(0.2f, 0.3f, 0.3f, 1);
		glClear(GL_COLOR_BUFFER_BIT);
		myShader.use(); // 激活程序对象
		//myShader.setFloat("offset", 0.5f); // 设置位置偏移量
		glBindVertexArray(VAO); // 这里只有一处使用可以省略
		glDrawArrays(GL_TRIANGLES, 0, 3); // 绘制三角形，起始索引为0，要绘制顶点数量为3

		glfwSwapBuffers(window);
		glfwPollEvents();
	}

	glDeleteBuffers(1, &VBO);
	glDeleteVertexArrays(1, &VAO);
	glfwTerminate();
}
```

我们的顶点数据还是前面的彩色三角形，只是把位置换到了中间，运行后的效果图如下：

![效果图](https://github.com/tizengyan/images/raw/master/opengl_colored_triangle2.png)

思考题中有一个在顶点shader用uniform定义一个偏移量，使三角形向右平移，我们只需要在源码中加入这个变量，然后将其作为x方向的偏移量与位置向量相加，然后在**激活程序对象之后**调用shader类的设置函数为其赋值就行了。

## 纹理（Textures）


