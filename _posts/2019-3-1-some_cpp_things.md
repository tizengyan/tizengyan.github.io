---
layout: post
title:  "C++的一些补充"
#date:   2019-03-01 8:10:54
categories: 基础知识
tags: C++
excerpt: 整理一下C++中可能会被问到的知识点
author: Tizeng
---

* content
{:toc}

## 1.整形的范围

C++中`int`类型为4个字节，也就是32bit，共有`2^32`次方种可能，由于有正负之分，一边一半，加上中间的0，因此它的范围是

> [-2^31, 2^31 - 1]

## 2.深拷贝和浅拷贝

### 浅拷贝

将一个已有对象给新创建的对象赋值时，除了原始类型（primitive type）外，其他字段均只拷贝引用，因此只要这两个对象中任意一个中的成员被改变，都会影响到另一个对象。这样做的好处是速度快，消耗少。

### 深拷贝

深拷贝则是在赋值时创建一个新的对象，并将之前对象的所有成员变量都在新对象中重新创建，这样新对象就与之前的对象相互独立，这样做消耗更大，需要分配新的内存空间。

我们可以将这两种拷贝模式结合起来，在一开始拷贝的时候只是进行浅拷贝，等到我们需要将拷贝的新对象做改动的时候再去分配新的空间创建与之前独立的对象。这样从效果看来和深拷贝一样，但是一定程度上提高了效率。

## 3.引用和指针

* 指针可以指向`NULL`，而引用始终指向某个变量或对象

* 可以对指针再取地址，而不可以对引用取地址，也就是说可以有多层级的指针，而引用只有一层

* 引用被创建的同时必须被初始化（指针则可以在任何时候被初始化）

* 指针可以更改指向的目标，引用一旦初始化后就不可更改

* 指针可以用自增或自减操作符移动到相邻的地址所指向的变量，而引用不行

* 指针取值需要加`*`关键字，而引用不需要

* 指针用`->`访问成员，引用用`.`

* 引用其实是它所关联对象的别名，对引用的任何操作都相当于直接对本体操作

### `void*`指针

这种指针可以指向任何类型的变量，但是不可以直接用`*`来取值，如果需要取值，则需要转换类型：

```c++
int a = 10;
void *ptr = &a;
printf("%d", *(int *)ptr);
```

### 指针数组

```c++
char (*a)[10];
```

这个语句表示声明一个数组`*a`，类型为`char`，因此`a`为指向这个数组的指针。

```c++
char *a[10];
```

而这个语句则声明的是字符指针类型的数组，大小为10，每个元素都是指向`char`的指针。

`malloc`返回的便是`void`指针，因此可以对任何类型的变量分配空间。

## 4.智能指针

## 5.隐式转换

所谓隐式转换，是指不需要用户干预，编译器私下进行的类型转换行为，一般会从低精度向高精度转换。常见情况有：混合类型计算表达式、函数传参、不同类型赋值、函数返回值等。

隐式转换在类的构造函数中可能造成意想不到的问题，为了避免可以在构造函数前加上`explicit`来禁止隐式转换。

## 6.枚举类型

用`enum`关键字来定义新的数据类型，

```c++
enum enumType {Monday, Tuesday, Wednesday, Thursday, Friday, Saturday, Sunday};
```

这里我们声明了`enumType`为新的数据类型，Monday、Tuesday等为符号常量，通常称之为枚举量，其值默认分别为 0-6。

```c++
enumType weekday;
weekday = enumType(2);
```

等同于

```c++
Weekday = Wednesday;
```

枚举量Monday、Tuesday等的值默认分别为 0-6，但我们可以显式的设置枚举量的值：

```c++
enum enumType {Monday=1, Tuesday=2, Wednesday=3, Thursday=4, Friday=5, Saturday=6, Sunday=7};
```

枚举量的值可以相同。

我们甚至可以用枚举类型的特点来声明特殊的常量：

```c++
enum { NumTurns = 5 };
int scores[NumTurns];
```

## 7. ++*p, *p++和 *++p

这里考察对运算符优先级的理解，前缀`++`的结合性是从右到左，而后缀`++`的结合性是从左到右，而后缀`++`比前缀`++`和`*`**优先级要高**，具体我们看下面的代码：

```c++
int arr[] = {10, 20};
int *p = arr;
++*p;
printf("arr[0] = %d, arr[1] = %d, *p = %d", arr[0], arr[1], *p);
```

这里`++*p`中，前缀`++`和`*`优先级相同，方向从右往左，可以改写为`++(*p)`，即先取值在自增，因此输出为：

> arr[0] = 11, arr[1] = 20, *p = 11

```c++
int arr[] = {10, 20};
int *p = arr;
*p++; 
printf("arr[0] = %d, arr[1] = %d, *p = %d", arr[0], arr[1], *p);
```

对`*p++`而言，后缀`++`比`*`优先级要高，可以改写为`*(p++)`，即把指针后移一位，然后取值，因此输出为：

> arr[0] = 10, arr[1] = 20, *p = 20

```c++
int arr[] = {10, 20};
int *p = arr;
*++p; 
printf("arr[0] = %d, arr[1] = %d, *p = %d", arr[0], arr[1], *p);
```

最后来看`*++p`，与第一种情况一样，前缀`++`和取值`*`优先级相同，但它们的方向都是从右往左，所以应该改写为`*(++p)`，得到输出：

> arr[0] = 10, arr[1] = 20, *p = 20

## 8.文件和流

常规输入输入`cout`和`cin`需要包含`iostream`头文件，而当我们需要从文件中读写数据时，就需要包含`fstream`头文件。具体看代码：

```c++
#include <iostream>
#include <fstream>

using namespace std;

int main(){
    ostream outFile; // 以写模式打开文件
    outFile.open("filename");
    int data;
    cin >> data;
    outFile << data << endl; // 写入数据
    outFile.close(); // 关闭文件

    ifstream inFile; // 以读模式打开文件
    inFile.open("filename");
    inFile >> data; // 读取文件的数据
    cout << data << endl; // 在屏幕上打印读取的数据
    inFile.close(); // 关闭打印的文件
    return 0;
}
```

## 9.typedef关键词

`typedef`会定义一种类型的别名，而不是像`#define`一样简单的替换字符串，举个简单的例子：

```c++
typedef char* charp2;
#define charp1 char*

int main() {
    charp1 p1, p2;
    charp2 p3, p4;
    cout << "type p1, p2: " << typeid(p1).name() << ", " << typeid(p2).name() << endl;
    cout << "type p3, p4: " << typeid(p3).name() << ", " << typeid(p4).name() << endl;
    system("pause");
    return 0;
}
```

输出：

> type p1, p2: char *, char
>
> type p3, p4: char *, char *

首先`#define`后面不能加分号，否则会把分号一起并入宏定义。然后可以看到由于只是单纯的替换字符串，定义指针时除了第一变量外其余的都没有定义成指针变量。

```c++
char s[] = "abcd";
const charp1 p1 = s;
const charp2 p2 = s;
p1++;
p2++; // compile error
```

可以看到这里`p2`并不是将被指向的对象设为常量，而是常量的指针，因此不可以对指针的指向做改动。

再举个例子：

```c++
typedef char T[10];
T * a;
```

问第二句话和`char (*a)[10]`还是`char *a[10]`等价。其中这两个语句的区别上面已经分析过，这里`typedef char T[10]`的意思是定义一个数据类型`T`，类型为`char[10]`，就是说`T *a`代表`a`是指向T类型的指针，也就是指向`char[10]`的指针，也就是`char (*a)[10]`。

## 10.extern关键词

用来在文件中重新声明在其他文件（模块）中定义的全局变量，或者我们可以直接包含定义了全局变量的头文件。