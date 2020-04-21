---
layout: post
title:  "LudumDare Gamejam开发笔记"
categories: 游戏开发
tags: gamejam
excerpt: 第一次参加gamejam，记一下开发中遇到的问题
author: Tizeng
---

* content
{:toc}

## hp存哪里

## 对象的生成顺序

Start方法中最好不要初始化重要变量，尤其是会被外部使用的变量，不然外部使用完后如果对象在之后初始化，就会把值置回初始值。

## 主界面ui逻辑

先想的是用reload场景来开始游戏，
unscaledTime

## 储存游戏data

跨场景

## 场景速度控制

从GM传速度到传速度递增率ratio

## 回调函数

## 脚本变量layout

## GamaManager职责

## Debug日志

1. StopCoroutine没有停止之前的coroutine

原因是之前调用StartCoroutine时传进去的是方法而不是方法名称的字符串，这样StopCoroutine会无法识别。也可以在start时存储coroutine的实例，然后stop的时候直接将这个实例传进去。

2. 主角多个collider重复触发damage


