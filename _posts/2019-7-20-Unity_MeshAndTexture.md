---
layout: post
title:  "Unity学习笔记9——Mesh & Texture"
#date:   2019-03-01 8:10:54
categories: 游戏开发
tags: unity
excerpt: Unity中的贴图与网格
author: Tizeng
---

* content
{:toc}

* mesh意为网格，可以理解为是3D模组，在GameObject上添加一个MeshFilter和一个MeshRender，然后将准备好的mesh导入进MeshFilter的话，就可以在Scene中显示出来模型的样子。

* texture意为贴图，本质上是一张图片，合理的将其以UV坐标附在模型上的话，就可以让模型有真实感。

## 合并网格

有时候为了优化或其他原因，需要将数个网格合并成一个网格，可以使用方法`CombineMeshes`

```c#
GameObject parent = new GameObject();

// ... ...

Mesh finalMesh = new Mesh();
MeshFilter[] meshFilters = parent.GetComponentsInChildren<MeshFilter>();
CombineInstance[] combines = new CombineInstance[meshFilters.Length];
for(int i = 0; i < meshFilters.Length; i++) {
    combines[i].mesh = meshFilters[i].sharedMesh;
    combines[i].transform = meshFilters[i].transform.localToWorldMatrix;
}
finalMesh.CombineMeshes(combines);
```

其中的输入`combines`类型为`CombineInstance`的数组，`parent`为包含了若干子物体的父物体，我们先将所有子物体的mesh拿到，赋给新建的`combines`数组。
注意调用MeshFilter中的mesh的时候要用`sharedMesh`而不是`mesh`，最后用`finalMesh`调用合并方法就可以将合并后的网格存入`finalMesh`。

如果我们要将新合成的网格赋给一个空GameObject，调用`AddComponent`往上加：

```c#
GameObject go = new GameObject(resMesh.name);

MeshFilter mf = go.AddComponent<MeshFilter>();
mf.sharedMesh = resMesh;

MeshRenderer mr = go.AddComponent<MeshRenderer>();
mr.sharedMaterial = sourceMaterial;

// 生成prefab
GameObject prefab = PrefabUtility.SaveAsPrefabAsset(go, "Assets/output.prefab");
prefab.GetComponent<MeshFilter>().sharedMesh = resMesh;
AssetDatabase.AddObjectToAsset(resMesh, prefab);

DestroyImmediate(go);
```

但是最后生成prefab时，由于上面的mesh是我们在代码中生成的，如果我们直接将这个GameObject变成模板，那么刚刚赋上的新mesh会丢失，因此保存为prefab后，还需要拿到它的引用再存一次。

## 合并贴图

Unity中一般说的贴图是Texture2D，它本质上一长串像素的颜色，但我们无法将它直接拖入游戏场景中，除非我们将其转化为一个Sprite。

合成的方法很简单，获得每个贴图的像素，然后依次贴在新的贴图上，至于是横向合并还是纵向合并看需求，最后不要忘了`Apply`，然后以Byte类型保存为png图片：

```c#
int height = tex1.height > tex2.height ? tex1.height : tex2.height;
int width = tex1.width + tex2.width;
Texture2D newTex = new Texture2D(width, height);
for(int j = 0; j < newTex.height; j++) {
    for(int i = 0; i < tex1.width; i++) {
        newTex.SetPixel(i, j, tex1.GetPixel(i, j));
    }
    for (int i = 0; i < tex2.width; i++) {
        newTex.SetPixel(i + tex1.width, j, tex2.GetPixel(i, j));
    }
}
newTex.Apply();

var bytes = ImageConversion.EncodeToPNG(newTex);
System.IO.File.WriteAllBytes("Assets/newTexture.png", bytes);
```
