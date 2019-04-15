---
layout: post
title:  "C++的一些补充2"
#date:   2019-03-01 8:10:54
categories: 基础知识
tags: C++
excerpt: 整理一下C++面试中可能会被问到的知识点
author: Tizeng
---

* content
{:toc}

## 1.lambda表达式

这是C++11中的新特性，又叫**匿名函数**（Anonymous function），通常用在行数特别少而且只使用一次的函数上，对于这种函数而言没有为它命名的必要，因此可以用lambda表达式代替，它的格式为：

```c++
[ capture clause ] (parameters) -> return-type { function_body }
```

其中返回值类型和前面的`->`可以被省略，圆括号为传入函数的参数声明，同时我们可以对外部的变量进行捕获，方括号内规定的就是捕获变量的名称以及是`by value`还是`by reference`，下面是[维基](https://en.wikipedia.org/wiki/Anonymous_function#C++_(since_C++11))中的定义：

    []        // no variables defined. Attempting to use any external variables in the lambda is an error.
    [x, &y]   // x is captured by value, y is captured by reference
    [&]       // any external variable is implicitly captured by reference if used
    [=]       // any external variable is implicitly captured by value if used
    [&, x]    // x is explicitly captured by value. Other variables will be captured by reference
    [=, &z]   // z is explicitly captured by reference. Other variables will be captured by value

### 应用场合

#### (1)在main执行前运行语句

看下面这段代码（来自知乎）：

```c++
int a = []() {
    std::cout << "a" << endl;
    return 0;
}();

int main() {
    cout << "b" << endl;
    system("pause");
    return 0;
}
```

控制台会先输出 ab ，也就是说在`main`运行前 a 就已经被输出了，这是因为在运行`main`之前，会先构造全局变量，如果此时将全局变量的赋值语句写成一个 lambda 表达式，就可以达到在`main`运行前就运行其他语句的目的，最后变量`a`会被赋值为0。

还有另一种方法，就是创建一个类的全局对象变量，然后在该类的构造函数中放进要执行的语句，也可以达到相同的目的：

```c++
class TestClass{
public:
    TestClass() {
        cout << "TestClass" << endl;
    }
};

TestClass Ts; // 定义个全局变量，让类里面的代码在main之前执行

int main(){
    cout<<"main"<<endl;
    return 0;
}
```

虽然我不知道这种操作有什么卵用，但是面试可能会考。

#### (2)在使用STL时简化代码

直接看代码：

```c++
vector<int> v {4, 1, 3, 5, 2, 3, 1, 7};

for_each(v.begin(), v.end(), [](int i) {
    std::cout << i << " ";
});

// 输出第一个大于4的元素
vector<int>:: iterator p = find_if(v.begin(), v.end(), [](int i) {
    return i > 4;
});
cout << "First number greater than 4 is : " << *p << endl;

// 按递增顺序排序数组
sort(v.begin(), v.end(), [](const int& a, const int& b) {
    return a > b;
});

// 计算容器中大于等于5元素的个数
int count_5 = count_if(v.begin(), v.end(), [](int a) {
    return (a >= 5);
});
cout << "The number of elements greater than or equal to 5 is : "
    << count_5 << endl;

// 去重，注意unique去重之后并不会对容器的大小进行调整，而是将目前最后的一个元素用迭代器的方式返回
p = unique(v.begin(), v.end(), [](int a, int b) {
    return a == b;
});

// 将容器的大小重新调整
v.resize(distance(v.begin(), p));
```

如果将上面每一次操作后的`v`输出会得到：

> 4 1 3 5 2 3 1 7
>
> First number greater than 4 is : 5
>
> 7 5 4 3 3 2 1 1
>
> The number of elements greater than or equal to 5 is : 2
>
> 7 5 4 3 2 1

#### (3)捕获外部变量

```c++
int t = 2;
auto test = [&t](int i) {
    t++;
    return t * i;
};
cout << "test of 5 is : " << test(5) << endl;
cout << "type of test is : " << typeid(test(5)).name() << endl;
cout << "t = " << t << endl;
```

此时`test(5)`会输出15，类型为`int`，而`t`会递增为3，因为是引用捕获。

奇怪的是这里如果把`test`声明时的类型`auto`改为`int`会出现一个很奇怪的错误（no suitable conversion function from lambda ...），谷歌之后感觉这个问题目前超出我的理解范围，先留个坑。