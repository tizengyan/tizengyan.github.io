---
layout: post
title:  "C++的各种转型（cast）"
#date:   2019-03-05 18:20:54
categories: 基础知识
tags: C++
excerpt: 整理一下C++中的各种转型问题
author: Tizeng
---

* content
{:toc}

## 左值和右值

左值指的是既能够出现在等号左边也能出现在等号右边的变量(或表达式)，右值指的则是只能出现在等号右边的变量(或表达式)。

## 转型（cast）

传统的转型有两种写法：

```c++
(T) expression;
T (expression);
// 都将expression转型为T
```

C++提供四种新式转型：`danamic_cast`、`static_cast`、`const_cast`、`reinterpret_cast`，它们各有不同的目的。

引进两个概念：上行转换（derived to base)，下行转换（base to derived）。

### *typeid*类型检查

`typeid`操作符返回一个`type_info`类型的常量对象的引用，这个值可以和另一个用`typeid`获取值的对象用`==`或者`!=`相比较，或者直接用`name()`成员获取类型名称（返回的类型名称字符串与编译器有关）。

下面是示例代码：

```c++
int main () {
    int * a,b;
    a=0; b=0;
    if (typeid(a) != typeid(b))
    {
    cout << "a and b are of different types:\n";
    cout << "a is: " << typeid(a).name() << '\n';
    cout << "b is: " << typeid(b).name() << '\n';
    }
    return 0;
}
```

输出：

> a and b are of different types:<br>a is: int *<br>b is: int

当`typeid`被用在多态类型中时，结果会返回那个最为具体和完整的对象类型（the result is the type of the most derived complete object）。

```c++
class CBase { virtual void f(){} };
class CDerived : public CBase {};

int main() {
    try {
    CBase* a = new CBase;
    CBase* b = new CDerived;
    cout << "a is: " << typeid(a).name() << '\n';
    cout << "b is: " << typeid(b).name() << '\n';
    cout << "*a is: " << typeid(*a).name() << '\n';
    cout << "*b is: " << typeid(*b).name() << '\n';
    } catch (exception& e) { cout << "Exception: " << e.what() << endl; }
    return 0;
}
```

输出：

> a is: class CBase * <br>b is: class CBase *<br>*a is: class CBase<br>*b is: class CDerived

当输入为空指针时，它会抛出一个`bad_typeid`异常。

### *static_cast*

用来强迫隐式转换，例如将一个`non-const`的对象转换为`const`对象，或将`int`转换为`double`等等。它也可以用来执行上述多种转换的反向转换，例如将`void*`指针转为typed指针，将pointer-to-base转为pointer-to-derived。但是他无法将`const`转为`non-const`，这个只有`const-cast`才能够办到。但它没有运行时类型检查来保证转换的安全性，比如若将4bytes的`int`用`static_cast`转换成1byte的`char`，只会保留前八位的数据，也就是说如果整形范围超过`[-128,127]`，就会发生数据丢失。

### *dynamic_cast*

`dynamic_cast`主要用来执行“安全向下转型”（safe downcasting），也就是用来决定某对象是否归属继承体系中的某个类型。即转换成功就返回转换后的正确类型指针，如果转换失败，则返回`NULL`，之所以说`static_cast`在下行转换时不安全，是因为即使发生不安全的转换，它也不返回`NULL`。

语法：

> dynamic_cast < new_type > ( expression )

* new_type - 必须是指向类的**指针**、类的引用，或`void`指针

* expression - 如果`new_type`为指针，则必须为指针；如果`new_type`为引用则为左值

若转型成功，则`dynamic_cast`返回`new_type`类型的值。若转型失败且`new_type`是指针类型，则它返回该类型的空指针。若转型失败且`new_type`是引用类型，则它抛出匹配类型`std::bad_cast`处理块的异常。

必须是基类指针本来就是指向一个子类，才能从基类指针转成子类指针，而且基类要保证是**多态类**，也就是要含有带`virtual`关键字的函数。

首先如果`p`指向的是**子类**对象，要把它转换成基类，则`dynamic_cast`和`static_cast`都可以转换成功，且不需要基类是多态类，如下所示：

```c++
Base *p = new Derived();
Derived *pd1 = static_cast<Derived *>(p);
Derived *pd2 = dynamic_cast<Derived *>(p);
```

但如果`p`指向的是**父类**对象，要把它转换成子类：

```c++
Base *p = new Base;
Derived *pd3 = static_cast<Derived *>(p); // 正常返回
Derived *pd4 = dynamic_cast<Derived *>(p); // 返回NULL
```

虽然不会报错，但这是不安全的，如果`pd3`调用了子类有而父类没有的成员，就会导致程序崩溃（此时基类**必须是多态类**）。而`dynamic_cast`能发现上述问题，将返回`NULL`。

总结，使用`dynamic_cast`时：

* 指向子类的基类指针转为指向基类，即向上转换，转换成功（不需要多态基类）

* 指向子类的基类指针转为子类，即安全的向下转换，转换成功（需要多态基类）

* 指向基类的基类指针转为子类，即不安全的向下转换，返回`NULL`（需要多态基类）

* 可以将任何类型的指针转换为`void*`

### *const_cast*

被用来将对象的常量性移除，即将`const`转为`non-const`，C++中只有`const_cast`才能做到。

### *reinterpret_cast*

这个转换方式用来将一种类型的指针转换成另一种类型的指针，不管它们是否相关，这是一个很不安全也没有意义的操作，但如果使用不当，`reinterpret_cast`会极易引发bug。

它唯一的保证是讲一个指针转化为`int`型时，只要`int`有足够大的空间容纳，就可以将它转化回那个指针。

## CV（const and volatile）类型限定

### mutable限定符

若一个类的成员被声明为`mutable`，那么即使含有它的对象被声明为`const`，我们也可以对它进行改动。声明时要注意保证为`non-static`类成员和`non-reference`以及`non-const`类型。

### `volatile`限定符

翻译为多变型、无常型，这种类型极少被使用，但有存在的必要，以如下代码为例：

```c++
int some_int = 100;

while(some_int == 100){
   // your code
}
```

编译器在编译的时候会发现`some_int`这个变量在程序中从未被修改，也就是说这个`while`循环是个死循环，编译器为了加快程序的性能，可能会将代码做出优化，将循环部分改成`while(true)`，这样一来在编译器眼中程序结果没有变化，但是效率变高了，而我们可能需要在这个程序外部对`some_int`进行更改，从而实现某些功能，但这时编译器的优化（optimizer）使这个程序出现了bug，为了防止这种情况发生，我们可以使用`volatile`限定符：

```c++
volatile int some_int = 100;
```

这样一来上面所说的bug就不会发生了。

## 代码示例

下面举一些`const`和`mutable`的例子：

```c++
int main(){
    int n1 = 0;           // non-const object
    const int n2 = 0;     // const object
    int const n3 = 0;     // const object (same as n2)
    volatile int n4 = 0;  // volatile object
    const struct{
        int n1;
        mutable int n2;
    } x = {0, 0};      // const object with mutable member

    n1 = 1; // ok, modifiable object
//  n2 = 2; // error: non-modifiable object
    n4 = 3; // ok, treated as a side-effect
//  x.n1 = 4; // error: member of a const object is const
    x.n2 = 4; // ok, mutable member of a const object isn't const

    const int& r1 = n1; // reference to const bound to non-const object
//  r1 = 2; // error: attempt to modify through reference to const
    const_cast<int&>(r1) = 2; // ok, modifies non-const object n1

    const int& r2 = n2; // reference to const bound to const object
//  r2 = 2; // error: attempt to modify through reference to const
//  const_cast<int&>(r2) = 2; // undefined behavior: attempt to modify const object n2
}
```

在类的成员函数中，要声明一个`const`类型的类成员函数，只需要在成员函数参数列表后加上关键字`const`，如：

```c++
int get() const;
```

若将成员函数声明为`const`，则：

1.该函数不允许修改类的任何数据成员，不管其是否具有`const`性质

2.只能访问`const`函数

3.`const`对象只能访问const成员函数

因此在声明一个成员函数时，若该成员函数并不对数据成员进行修改操作，应尽可能将该成员函数声明为`const`成员函数。