---
layout: post
title:  "C#基础"
#date:   2019-03-06 12:12:54
categories: C#
tags: C#
excerpt: 总结一下C#的一些基础知识
author: Tizeng
---

* content
{:toc}

为了更好的搬砖，查了一些C#的资料，记录在此，方便以后回忆和整理，大部分内容为《C#图解教程》的笔记。

## 值类型和引用类型

值类型数据储存在栈中，只需要一段内存，而引用类型需要两段内存，第一段是**数据**储存在堆中，第二段是**引用**储存在栈中。对于引用类型的对象，它的所有数据都在堆中，不管它是否有值类型的成员。引用类型包括object、string、dynamic、class、interface、delegate、array。

函数参数中还有一种输出参数，声明和调用时都用`out`修饰符修饰，它和被`ref`修饰的参数类似，都让形参与实参指向同一块内存，但输出参数必须在使用前赋值，这意味着参数在传入时不需要初始化。

## 类

### 类的静态成员

和C++一样，C#中的静态成员独立于类的实例，在堆中会有一块专门的区域储存，对于所有类的实例而言，静态成员只有一个副本，因此所做的任何改动都会影响所有实例。访问时直接用类的名称加点运算符（.）。
即使没有实例，也可以访问静态成员。

非静态成员称为实例成员，**静态函数成员**不能访问实例成员，但可以访问其他静态成员。

常量数据不能被声明为静态的。但它的表现得和静态值一样，即使没有实例，依然可以访问。

### 静态构造函数

通常，静态构造函数初始化类的静态字段。与实例构造函数不同的是，类只能有一个静态构造函数，而且不能带参数，也不能有访问修饰符。同静态方法，静态构造函数不能访问类的实例成员，因此也不能使用`this`访问器。

我们也不能从程序中显式调用静态构造函数，下面两种情况时，系统会自动调用它们：

* 类的任何实例被创建之前

* 类的任何静态成员被引用之前

### 类的属性

这是C#中常用的一种写法。我们可以像操作字段一样操作属性，而本质上属性是一个**函数成员**，声明语法如下：

```c#
class C1{
    private int val; //后备字段
    public int MyVal{
        set{
            val = value;
        }
        get{
            return val;
        }
    }
}
```

上面的代码声明了一个名为`MyVal`的`int`型属性，其中set和get是被称为访问器的方法，set访问器有一个单独的隐式的值参，名为`value`，与属性的类型相同，返回类型为void。get访问器没有参数，返回类型与属性类型相同。

要注意的是属性本身并没有任何存储，由访问器决定如何处理发进来的数据。定义好后，我们就可以像访问普通字段一样访问或赋值这个属性了，属性会根据赋值还是读取，隐式的调用访问器。

为了方便理解，一般来说属性和其后备字段有几种命名约定，一种是字段使用Camel大小写（myVal），属性使用Pascal大小写（MyVal）；第二种与第一种类似，只是字段前面加一个下划线（_myVal）。

属性和字段一样可以被声明为`static`。

#### 属性的优势

get访问器为只读属性，set访问器为只写，在定义属性时我们可以不定义某个访问器。因此属性可以只读或只写，而字段不行。

属性可以输入和输出，比如可以将set访问器中的代码改成`val = value > 100 ? 100 : value;`。

有一种常见的封装写法如下：

```c#
public string Name {get; private set;}
```

这样一来，在外部就只能读取该属性，而不可以设置。

有几点要注意：

* 两个访问器中只能有一个有访问修饰符

* 访问器的访问修饰符必须必成员的访问级别有更严格的限制性。比如如果一个属性的访问级别是public，那么访问器声明为任意一个非public级别，而如果属性的访问级别为protected，那么唯一能对访问器使用的修饰符就只有private了。

### this关键字

`this`只能被用在：实例构造函数、实例方法、属性和索引器的实例访问器中，它返回当前实例的引用。

## 结构（struct）

结构与类十分相似，不同的是结构为值类型，且不能被派生。结构生成的实例的成员储存在栈中，因此把一个结构赋值给另一个结构它们的成员会相同，但类则栈中的两个实例指向堆中同一个对象。

结构没有析构函数一说，其无参数的构造函数由编译器隐式提供，且不能被重载，它会将所有成员设为默认值（引用成员为null），只有用new运算符生成实例才会调用构造函数，若未用new创建实例，则必须显示的初始化数据成员后才能使用它们，在对所有数据成员赋值后才能调用任何函数成员。

## 索引器

索引器是一组get和set访问器，与属性类似，不用分配内存，不同的是，属性通常表示单独的数据成员，索引器通常表示多个数据成员。

使用索引器，还需注意以下几点：

* 索引器可以只有一个访问器，也可以两个都有

* 索引器总是实例成员，因此不能被声明为`static`

* 和属性一样，set和get访问器并不一定要关联某个字段

* 索引器没有名称，在名称的位置是关键字`this`

* 参数列表在方括号中间，且至少声明一个参数

下面看实例代码：

```c#
class C1{
    int temp0, temp1;
    public int this [int index]{
        get{
            return 0 == index ? temp0 : temp1;
        }
        set{
            if(0 == index)
                temp0 = value;
            else
                temp1 = value;
        }
    }
}
```

### 索引器重载

只要索引器的参数列表不同，类就可以有任意多个索引器。只是返回类型不同是不够的。

## 委托（delegate）

委托可以被看作是持有一个或多个方法的对象，但委托可以被执行，它会按顺序执行所有持有的方法，不论方法是否重复：

```c#
delegate void MyDel(int val); // 声明委托类型
class Program{
    void f1(int val){
        //...
    }
    void f2(int val){
        //...
    }

    static void main(){
        Program program = new Program();
        Mydel del; // 委托变量
        del = new Mydel(myInstObj.M1); // 创建委托对象并赋值实例方法
        del = SClass.OtherM2; // 创建委托对象并赋值静态方法（省略new），覆盖M1
        del += myInstObj.M2;  // 增加M2
        del -= SClass.OtherM2; // 删除OtherM2
        del(42); // 调用委托
    }
}
```

委托变量接收的方法类型必须和声明的委托一样，每次对委托进行赋值或增加、删减方法时都在内存中创建了新的委托对象，然后将委托变量指向它，因为委托对象一旦创建就不可被改变。从委托中删减方法时`-=`运算符将从调用列表最后开始搜索，并移除第一个与之匹配的实例，若要删除的方法不存在则无事发生，但调用空委托会抛出异常，当调用列表为空时委托为null，因此调用前可以与null进行比较。

### 返回值

委托的返回值（如果有）为调用列表中最后一个方法的返回值，其他方法的返回值会被忽略。

### 引用参数

如果委托的参数中有引用参数，该参数可能在调用列表中的每个方法里被改变，在调用下一个方法时，传入的参数也是被改变了的，和C++一样，引用即是变量的别名，操作的是同一段内存中的数据，不同的是C#中用关键字`ref`声明引用而不是`&`，而且传入时同样要加上`ref`。

### 匿名方法

有时注册给委托的方法只会被使用一次，这时就可以用匿名方法来创建委托：

```c#
delegate int Foo(int x);
static void Main(){
    int y = 10;
    Foo foo = delegate(int x) {
        return x + y; // 使用外部变量 
    };
}
```

如果匿名方法的参数列表不包含out参数，方法内部也没有使用参数，那么delegate后的括号和参数可以被省略。匿名方法中可以使用外部变量，称之为捕获，被捕获的变量即是在外部已经离开作用域，在匿名方法内也会一直有效，这称为对其生命周期的扩展。

C#3.0引入了Lambda表达式，进一步简化了匿名方法的语法：

```c#
MyDel del = delegate(int x) { return x + 1; };
MyDel le1 = (int x) => { return x + 1; }; // 省略delegate
MyDel le2 = x => { return x + 1; }; // 省略括号和参数类型
MyDel le3 = x => x + 1; // 省略return
```

在使用上面写法的时候要注意：

* 委托有ref或out参数时不得省略参数类型
* 有多余一个参数或没有参数时不得省略括号
* Lambda表达式的签名必须与声明的委托一致（参数数量、类型、顺序、修饰符）

### unity中的委托

## 事件（Event）

事件是发布者/订阅者模式的基础，发布者定义一个或多个事件，如果其他程序或类感兴趣就可以订阅（注册），这样在事件发生的时候发布者就会触发订阅者注册的若干方法，也就是回调。事件包含了一个私有的委托，它对外部来说不可见，我们对事件来添加、删除或调用回调方法，实际上是通过内部的委托来实现，事件声明的时候也需要指定委托类型，注册的处理程序的签名类型必须和该委托类型相同，返回类型也要匹配。事件是类或结构的成员，因此必须声明在类或结构中：

```c#
delegate void Handler();

class C1 {
    public event Handler myEvent; // 事件声明在类中

    public void TriggerEvent() {
        if (myEvent != null)
            myEvent();
    }
}

class C2 {
    public int Count { get; private set; }

    public C2(C1 c) { // constructor
        Count = 0;
        c.myEvent += CountNum;
    }

    public void CountNum() {
        Count++;
    }
}

class Program {
    static void main() {
        C1 c1 = new C1();
        C2 c2 = new C2(c1);
        c1.TriggerEvent();
        Console.WirteLine("Count = {0}", c2.Count);
    }
}
```

事件声明后使用`+=`和`-=`来对其增加或删减方法，然后需要触发时像函数那样调用就行了。EventHandler是C#自带的一个专门处理系统事件的委托，称之为标准事件，它的返回类型是void，有两个参数，第一个是触发事件对象的引用，类型为object，因此可以匹配任何类型的实例，第二个参数类型为EventArgs，用来保存状态信息，也可以传递数据，但是需要声明一个派生自EventArgs的类来保存数据，这样做的好处是统一了委托的签名和返回类型，让所有的事件都可以使用，而不是为不同参数声明多种事件。下面是使用EventHandler的例子：

```C#
public class MyEventArgs : EventArgs {
    public int IterationCount { get; set; }; // 迭代次数
}

class C1 {
    public event EventHandler<MyEventArgs> myEvent; // 事件此时要声明为泛型

    public void TriggerEvent() {
        MyEventArgs args = new MyEventArgs();
        for(int i = 0; i < 100; i++) {
            if (i % 12 == 0 && myEvent != null) {
                args.IterationCount = i;
                myEvent(this, args);
            }
        }
    }
}

class C2 {
    public int Count { get; private set; }

    public C2(C1 c) { // constructor
        Count = 0;
        c.myEvent += CountNum;
    }

    public void CountNum(object source, EventArgs e) {
        Console.WriteLine("{0} iterations in {1}", e.IterationCount, source.ToString());
        Count++;
    }
}

class Program {
    static void main() {
        C1 c1 = new C1();
        C2 c2 = new C2(c1);
        c1.TriggerEvent();
        Console.WirteLine("Count = {0}", c2.Count, );
    }
}
```

如果需要自己定义`+=`和`-=`运算符，可以在声明事件时定义访问器add和remove，它们和属性类似，有隐式的value参数，接受实例或静态方法的引用，然后使用`+=`或`-=`时就会执行访问器内的代码，不过这需要我们自己实现存储和移除事件的方法，因为此时事件中已不包含任何内嵌委托对象了，这部分书上没有具体例子（鸽）。
