---
layout: post
title:  "C++基础"
#date:   2019-03-01 8:10:54
categories: C++
tags: C++
excerpt: 整理一下C++中可能会被问到的知识点
author: Tizeng
---

* content
{:toc}

## 1.整形的范围

C++中`int`类型为4个字节，也就是32bit，共有$2^{32}$次方种可能，由于有正负之分，一边一半，加上中间的0，因此它的范围是$[-2^{31}, 2^{31} - 1]$。

### double的范围和精度

c++中双精度浮点数至少可以精确到10位有效数字，一般来说是16位有效数字左右，微软官方文档中给出的范围是$1.7E +/- 308(15 digits)$。

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

`malloc`返回的便是`void`指针，因此可以对任何类型的变量分配空间。

### 指针与数组

```c++
char (*a)[10];
```

这个语句表示声明一个数组，类型为`char`，`a`为指向这个数组的指针。

```c++
char *a[10];
```

而这个语句则声明的是字符指针类型的数组，大小为10，每个元素都是指向`char`的指针。

* 只有指针数组，没有引用数组
* 当使用数组时，编译器实际上给我们的是指向第一个元素的指针
* 把指向数组第一个元素的指针加上数组size得到刚好位于最后一个元素再后一位的指针，可以表示类似iterator中的end成员，但一旦再超过一位就会越界
* 可以用`[]`对指针进行索引，拿到指向前后若干位元素的指针，索引可以为负，如`p[-1]`与`*(p-1)`等价，但要注意不能越界

## 4.重载运算符

重载的运算符是带有特殊名称的函数，函数名是由关键字 operator 和其后要重载的运算符符号构成的。与其他函数一样，重载运算符有一个返回类型和一个参数列表。

```c++
Box operator+(const Box& b){
    Box box;
    box.length = this->length + b.length;
    box.breadth = this->breadth + b.breadth;
    box.height = this->height + b.height;
    return box;
}
```

声明加法运算符用于把两个 Box 对象相加，返回最终的 Box 对象。大多数的重载运算符可被定义为普通的非成员函数或者被定义为类成员函数。如果我们定义上面的函数为类的非成员函数，那么我们需要为每次操作传递两个参数，如下所示：

```c++
Box operator+(const Box& b1, const Box& b2);
```

## 5.隐式转换

所谓隐式转换，是指不需要用户干预，编译器私下进行的类型转换行为，一般会从低精度向高精度转换。常见情况有：混合类型计算表达式、函数传参、不同类型赋值、函数返回值等。

隐式转换在类的构造函数中可能造成意想不到的问题，为了避免可以在构造函数前加上`explicit`来禁止隐式转换。

## 6.枚举类型

用`enum`关键字来定义新的数据类型：

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

## 7.运算符优先级

C++中的运算符优先级如下图所示，数字越小优先级越高，要特别注意的是结合性不同的运算符会有所不同：

![operator_priority](https://github.com/tizengyan/images/raw/master/operator_priority.png)

### ++*p, *p++和 *++p

这里考察对运算符优先级与结合性的理解，前缀`++`的结合性是从右到左，而后缀`++`的结合性是从左到右，而后缀`++`比前缀`++`和`*`**优先级要高**，具体我们看下面的代码：

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

### 输入流

除了`cin`之外，如果接收的是字符，还可以用`get`和`getline`，尤其是需要接收空格的时候。

```c++
string str;
getline(cin, str);
```

这种写法最简洁，不用指定长度并且会接收空格。其他写法如下：

```c++
char s1[20];
char s2[20];
cin.getline(s1, 20);
cin.get(s2, 20);
```

注意圆括号中的数字是要抓取的字符数量，这个数量一定不能超过前面的`s`，否则会越界导致程序崩溃。

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

补充一点关于typedef的实质，C++中变量的声明分为基本类型（base type）和说明符（declarator）两部分，比如整型指针`int *p`，基本类型是int，说明符是`*`以及变量名p，如果typedef了一个某某指针，那么指针就变成了这个类型的基本类型，会被const修饰：

```c++
typedef char *pstring;
const pstring cstr = 0; // cstr是指向char的const指针，而不是指向const char的指针
const pstring *ps; // ps是指针，指向一个const类型的char指针
```

### 左右法则

读声明的时候如果声明内容太过复杂，需要用到左右法则：从变量名开始，往右读，如果碰到圆括号则调转方向，圆括号内的内容读完后跳出括号继续，直到读完整个声明。

以下面这个声明为例：

```c++
int (*func)( int *p, int (*f)(int*));
```

从func开始往右读，遇到圆括号转向，看到*，说明是指针变量，出圆括号再次见到圆括号，说明func指向的是函数，第一个参数为int指针，第二个参数的读法再次从变量名开始，首先看到f，它是指针，出括号又看到括号，说明f也是函数指针，参数为int指针，因此func是一个参数为int指针和函数指针，返回类型为int的函数指针。

再看一个声明：

```c++
int (*func[5])(int *p);
```

还是从func开始，往右看到[5]，说明func是一个数组变量，然后遇到括号转向，看到*，说明这个数组中的元素是指针（这个问题前面已经讨论过），第一个括号中的内容结束，出来又看到括号，说明元素是函数，参数为int类型的指针，最后往左看知道返回类型是int，因此func是一个储存了5个函数指针的数组，它们（函数指针）指向的函数返回类型为int，参数为int指针。这个法则同样适用于引用指针的声明：

```c++
int i = 42;
int *p;
int *&r = p; // 从右往左读，r是引用，类型是整型指针
r = &i; // r是p的引用，也就是让p指向i
```

## 10.extern关键词

用来在文件中重新声明在其他文件（模块）中定义的全局变量，或者我们可以直接包含定义了全局变量的头文件。

C++中会区分声明（declaration）和定义（definition），声明是告诉程序一个名字，而定义则创建了一个与之关联的实体（entity），一个变量的声明确定了它的类型和名字，一个变量的定义同时也是它的声明，不同的是定义会给其分配空间，而且可能将变量初始化，使用`extern`关键词可以得到一个声明但是还未定义的变量：

```c++
extern int i; // declares but does not define i
int j; // declares and defines j
```

但如果我们显示的初始化了i，那么不管有没有extern它都是一个定义。变量的定义自始至终都只能有一次，但是声明可以有多次，例如我们需要在多个不同文件中使用同一个变量，那么这个变量必须只能在一个文件中被定义，其他使用了这个变量的文件必须声明，但不定义它，这就是为什么需要使用extern（《C++ Primer 5th》p.45）。

## 11.union

简单来说就是将同一块内存在需要时用于不同的用途，所占空间由最大成员所占用的空间决定，如：

```c++
union S {
    uint16_t n1; // 2 bytes
    uint32_t n2; // 4 bytes
    char c;      // 1 byte
};               // 4 bytes
S s;
s.n1 = 65;
cout << s.c << endl; // output A
cout << sizeof(s) << endl; // output 4
```

上面的例子中union中最大的成员占4字节，所以最后s会占四字节，声明n1为65后，此时s.n2和s.c没有被初始化，但读取时会读取s.n1设置的值65，因此输出c为字符A。再看一个匿名union的例子：

```c++
struct Node {
	union {
		int num;
		struct {
			short x, y;
		};
	};
};
Node node;
node.x = 32;
node.y = 22;
cout << "num = " << node.num << endl;   // 1441824
cout << (22 << 16) + 32 << endl;        // 1441824
```

由于xy成员为short类型，只占两个字节，num为int占四个字节，Node就占四字节，如果我们只初始化x和y，这四个字节中的前两个字节就会保存y，后两个字节保存x，输出num就是直接读取这四个字节中的所有数据的结果了。注意这里Node内部的union和struct都是匿名的，也就是说实例化Node后可以直接拿到union中的成员而不需要经过一道中间名。似乎这是C++11为了兼容C的写法而加入的特性，对于老一些的C++编译器可能并不允许这种写法。

## 12.emplace

C++为容器操作提供的新标准，这个词的意思是妥善放置，可以方便我们给容器加入元素，例如vector的push_back操作时不能直接在括号中用超过一个参数的构造器推入元素，而emplace_back可以：

```c++
class T {
public:	
    int i, j, k;
    T(int _i, int _j, int _k): i(_i), j(_j), k(_k) {}
};
vector<T> v;
T t(1, 2, 3);
v.push_back(t);
v.push_back(1, 2, 3);    // error
v.emplace_back(1, 2, 3); // ok
```

除了emplace_back之外，还有emplace_front和emplace，分别对应了push_front和insert。

## 13.make_heap

std::make_heap、std::push_heap、std::pop_heap都接收两个random-access iterator作为begin和end的参数，将这段内存中的数据构建成最大堆，并支持推入新元素或pop最大的元素，pop_heap会将最大元素放到end-1处，但并不会改变容器的大小，需要手动删除。
