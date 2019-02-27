---
layout: post
title:  "字符串的常见套路"
date:   2019-02-25 10:10:54
categories: 考点
tags: string
excerpt: 整理一下字符串的一些套路
author: Tizeng
---

* content
{:toc}

## 转换成字符数组

很多函数需要我们输入的是一个`char*`的字符数组而非`string`，这时我们就需要将二者相互转换的方法。

### string转char*

第一种可行的方法是使用`c_str()`，它会返回一个`const char*`。

```c++
std::string str = "string";
const char* cstr = str.c_str();
```

第二种是使用`data()`，并将结果转换为`char*`：

```c++
string str = "abc";
char* p = (char*)str.data();
```

### char*转string

如果我们需要将`char*`转换成`string`，可以利用`string`的构造函数：

```c++
const char *s = "Hello, World!";
string str(s);
```

## 字符串比较

这是很常规的一个操作，使用`strcmp`函数实现，它依次比较两个字符串的字符，返回值有三种可能：

* `<0`: the first character that does not match has a lower value in ptr1 than in ptr2

* `0`: the contents of both strings are equal

* `>0`: the first character that does not match has a greater value in ptr1 than in ptr2

注意这里输入的应该是`char*`。例如：

```c++
bool comp(Info info1, Info info2) {
    // 先按分数排序
    if (info1.score != info2.score)
        return info1.score > info2.score;
    char* a = (char*)info1.name.data();
    char* b = (char*)info2.name.data();
    // 再按名字字母顺序排序
    return strcmp(a, b);
}
```

## 子串

使用`std::string::substr`在字符串中提取子串，该函数定义为`string substr(size_t pos, size_t len) const;`。下面是用法：

```c++
int main () {
    std::string str="We think in generalities, but we live in details."; // (quoting Alfred N. Whitehead)
    std::string str2 = str.substr (3,5);     // "think"
    std::size_t pos = str.find("live");      // position of "live" in str
    std::string str3 = str.substr (pos);     // get from "live" to the end
    std::cout << str2 << ' ' << str3 << '\n';
    return 0;
}
```

输出：

> think live in details.

## 排序

如果我们需要把字符串中的字符按我们想要的顺序排序，可以直接用`sort`。

## 反转

顾名思义，将输入的字符串完全反向，然后输出。如`abcd`输出`dcba`。

### 反转一个整数

#### 转换成字符串实现

我们可以用翻转字符串的思路来实现，先将输入的整数转换成字符串，用翻转字符串的方法翻转过后再转换回整数。
整数转换成字符串可以直接用`to_string`函数，当然还有其他方法：

使用`itoa()`：

```c++
int a = 10;
char *intStr = itoa(a);
string str = string(intStr);
```

或者使用字符串流：

```c++
int a = 10;
stringstream ss;
ss << a;
string str = ss.str();
```

#### 直接对整数操作实现

思路很简单，初始化`sum = 0`，通过对输入的整数对10取余得到个位数，加上`sum * 10`，然后除以10抹掉最后一位，重复上述操作，直到取到0为止。

```c++
int reverse(int x) {
    long long temp = (long long)x;
    if(x < 0)
        temp = -1 * temp;
    long long res = 0;
    while(temp){
        long long dig = temp % 10;
        res = res * 10 + dig;
        temp /= 10;
    }
    // 如果题目要求答案范围不超出+-2^31
    if(res > pow(2,31) || res < -pow(2, 31) + 1)
        return 0;
    return x > 0 ? res : res * -1;
}
```

### 反转字符串

```c++

```

## 旋转字符串（或数组）

旋转分左旋和右旋，说简单点就是左移和右移，每次移动最后一位给到开头补齐，

字符串：

```c++

```

数组：

```c++

```

## 翻转句子中单词的顺序

即将输入的一句英文中的单词顺序反向后输出，例如`I am a student.`翻转之后变成`student. a am I`。

```c++

```
