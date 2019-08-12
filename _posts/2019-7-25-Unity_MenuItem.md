---
layout: post
title:  "Unity学习笔记8——自定义菜单和窗口"
#date:   2019-03-01 8:10:54
categories: 游戏开发
tags: unity
excerpt: Unity中用脚本定义的各种窗口
author: Tizeng
---

* content
{:toc}

## MenuItem

在脚本中的格式如下：

```c#
[MenuItem("MyMenu/Do Something")] // 根据某些规则后面还可以加上快捷键组合，这里暂且不提
static void DoSomething()
{
    Debug.Log("Doing Something...");
}
```

函数必须为`static`，这样就可以把函数的功能关联到菜单栏“MyMenu”中的下拉选项“Do Something”上。

## GetWindow

此方法用来创建可视化的窗口，和上面的MenuItem一起用就可以创造一个创建窗口的菜单按钮，有几种不同的写法，先介绍官网的写法：

```c#
[MenuItem("Lecture/Show Window")]
static void ShowWindow() {
    //var win = GetWindow<LectureTool>();
    //win.Show();
    EditorWindow.GetWindow(typeof(LectureTool), false, "My Window");
}
```

其中注释部分是另一种写法，拿到window后还要用`Show`函数显示。这样就可以创建出一个新的unity窗口了。

## OnGUI

这个函数会在处理GUI时被调用，我们可以用它来在前面生成的窗口中增加按钮等交互信息。下面是一个完整的脚本：

```c#
using UnityEngine;
using UnityEditor;

public class GUITest : EditorWindow {

    private void OnGUI() {

        if (GUI.Button(new Rect(10, 10, 50, 50), "Click1"))
            Debug.Log("Clicked the button with an image");

        if (GUI.Button(new Rect(10, 70, 50, 30), "Click2"))
            Debug.Log("Clicked the button with text");
    }

    [MenuItem("Test/Test")]
    static void Test() {
        EditorWindow.GetWindow(typeof(GUITest), false, "123");
    }
}
```

这个脚本不用挂在任何物件上，它会直接生效，我们会在unity的菜单栏看到多出的一个选项Test，

![gui1](https://github.com/tizengyan/images/raw/master/gui1.png)

以及它的下拉菜单中的选项，

![gui2](https://github.com/tizengyan/images/raw/master/gui2.png)

这些函数可以让我们方便的为unity开发扩展的菜单和功能。

## EditorGUILayout.ObjectField

在窗口中创建一个可以载入物件的接口，让用户可以从外部附上物件：

```c#
Mesh sourceMesh;
Material sourceMaterial;

sourceMesh = (Mesh)EditorGUILayout.ObjectField("Source Mesh", sourceMesh, typeof(Mesh), false);
sourceMaterial = (Material)EditorGUILayout.ObjectField("Source Material", sourceMaterial, typeof(Material), false);
scaleFactor = EditorGUILayout.Slider("Scale Factor", scaleFactor, 0.1f, 5); // 创建一个滑动条，总长为5，步长0.1
```

![gui3](https://github.com/tizengyan/images/raw/master/gui3.png)
