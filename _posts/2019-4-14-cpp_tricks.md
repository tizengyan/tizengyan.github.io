---
layout: post
title:  "《Effective C++》笔记（待完成）"
#date:   2019-03-01 8:10:54
categories: C++
tags: STL
excerpt: 整理一下《Effective C++》的笔记
author: Tizeng
---

* content
{:toc}

整理一下《Effective C++》的笔记。

## 尽量用const和enum替换#define（条款2）

如果需要定义类内部的常量，并保证其唯一性，可以定义为：

```c++
class GamePlayer{
private:
    static const int NumTurns = 5;
    int scores[NumTurns];
    ...
};
```

而如果编译器不允许内部初始化static成员，就只能在外部用`::`初始化，而如果此时我们又要在编译期间用到这个常量，可以用`enum`来补偿：

```c++
class GamePlayer{
private:
    enum { NumTurns = 5 };
    int scores[NumTurns];
    ...
};
```

另一方面，应该用下面的模板写法代替宏定义：

```c++
// 令人窒息的写法
#define CALL_WITH_MAX(a, b) f((a) > (b) ? (a) : (b))

// 正确的写法
template<class T>
inline void callWithMax(const T& a, const T& b){
    f(a > b ? a : b); // 较大者调用函数f
}
```

## 尽量少做转型动作（条款27）

