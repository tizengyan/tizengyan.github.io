---
layout: post
title:  "设计模式基础（待完成）"
categories: 基础知识
tags: design_pattern
excerpt: 总结一下设计模式的基本概念和常用的一些设计模式
author: Tizeng
---

* content
{:toc}

面向对象有三大基本特征，分别是是**封装**、**继承**和**多态**。

## 耦合性和内聚性

**耦合性**描述软件中各模块相互联系的紧密程度，模块之间联系越紧，其耦合度越强，独立性越差。

* 耦合性分类(低—>高)：无耦合、消息耦合、数据耦合、标记耦合、控制耦合、公共耦合、内容耦合。

**内聚性**指的是模块内各个元素彼此联系的紧密程度，一个模块内的元素联系越紧密，它的内聚性就越高。

* 内聚性分类(低–>高)：偶然内聚、逻辑内聚、时间性内聚、程序内聚、通信内聚、顺序内聚、功能内聚。

设计软件时要尽量做到，高内聚低耦合，提高模块的独立性。

紧密耦合的系统在开发阶段有以下的缺点：

1. 一个模块的修改会产生涟漪效应，其他模块也需随之修改。
2. 由于模块之间的相依性，模块的组合会需要更多的精力及时间。
3. 由于一个模块有许多的相依模块，模块的可复用性低。

## 设计模式的六大原则

1. 单一职责原则（Single Responsibility Principle）

    意思是一个类只应该负责一个职责。如果一个类负责多个职责，在对其中一个职责做修改时，可能会导致其他的职责出现问题，造成不必要的麻烦。应该将各个功能拆分成不同的类，提高内聚性。

2. 开闭原则（Open Close Principle）

    意思是对扩展开放，对修改关闭。在软件的生命周期内，因为变化、升级和维护等原因需要对软件原有代码进行修改时，可能会给旧代码中引入错误，也可能会使我们不得不对整个功能进行重构，并且需要原有代码经过重新测试。当软件需要变化时，尽量通过**扩展**软件实体的行为来实现变化，而不是通过修改已有的代码来实现变化。

3. 里氏替换原则（Liskov Substitution Principle）

    此原则强调，任何基类出现的地方，都可以被子类替换，而软件的功能不受影响。即基类派生子类时，可以增加新的功能，但尽量不要重写或重载基类的（非抽象）方法。换句话说子类可以对基类进行扩展，但不能改变基类原有的功能。

4. 依赖倒转原则（Dependence Inversion Principle）

    此原则定义为，高层模块不应依赖低层模块，二者都应依赖接口，细节应该依赖抽象。

    举例来说，我们需要一个读者读书，如果设计成读者一个类，书一个类，然后读者对象调用书的对象来读，这就是一个不符合该原则的设计，因为若要修改需求让读者读杂志、报纸等，需要依次对读者（上层结构）进行修改，这样加大了出错的风险，原因是耦合度太高，读者直接与书进行了互动。正确的做法是定义一个接口“读物”，读者与读物互动，不论输入的是何种读物，读者都可以阅读，而我们需要什么读物，直接继承这个接口实现就行了，即读者和书、报纸等，都依赖接口，对读物的改变不会影响读者这个类。

5. 接口隔离原则（Interface Segregation Principle）

    这个原则说的是一个类不应该依赖它不需要的接口，即一个类对另一个类的依赖应该建立在最小的接口上。如果接口的设计过于臃肿，让继承它的类不得不实现不需要的方法，那么就需要对这个接口进行拆分。

6. 迪米特法则，又称知识最少原则（Demeter Principle/Least Knowledge Principle）

    一个对象应该对其他对象保持最小的了解。类与类的关系越密切，耦合度就越大，我们应始终降低类与类之间的耦合。

补充一个合成复用原则（Composite Reuse Principle），这个原则让我们尽量使用合成/组合/聚合，少用继承。通过继承来复用的主要问题是会破坏类的封装，将基类的实现暴露给子类，如果基类发生改变，子类也必定会改变。如果新对象的某些功能在别的已经创建好的对象里面已经实现，那么尽量使用别的对象提供的功能，使之成为新对象的一部分，而不要自己再重新创建，这是降低耦合性的一种手段。

## 常见设计模式介绍

设计模式可以分为三大类，分别是

1. 创建型模式：

    单例模式(Singleton) 构建模式(Builder) 原型模式(Prototype) 抽象工厂模式(Abstract Factory) 

    工厂方法模式(Factory Method)

2. 结构型设计模式：

    装饰者模式(Decorator) 代理模式(Proxy) 组合模式(Composite) 桥连接模式(Bridge) 

    适配器模式(Adapter) 蝇量模式(Flyweight) 外观模式(Facade)

3. 行为型模式：

    策略模式(Strategy) 状态模式(State) 责任链模式(Chain of Responsibility) 解释器模式(Interpreter) 

    命令模式(Command) 观察者模式(Observer) 备忘录模式(Memento) 迭代器模式(Iterator) 模板方法模式(Template Method) 访问者模式(Visitor) 中介者模式(Mediator)

### 0.MVC模式

全称Model-View-Controller（模型-视图-控制器）模式，这种模式用于应用程序的分层开发。

### 1.单例模式

1. 单例模式的类始终只能生成一个实例

2. 单例类必须自己创建这个唯一的实例

3. 必须提供访问该实例的全局访问点。

实现单例有两种写法，懒汉和饿汉。

懒汉版（Lazy initialization）：

```c++
class FileSystem {
public:
    static FileSystem& instance() {
        // 又称惰性初始化
        if (instance_ == NULL)
            instance_ = new FileSystem(); // 两个线程可能先后进入这里实例化两次
        return *instance_;
    }

private:
    FileSystem() {}
    static FileSystem* instance_;
};
```

有一个问题，这种写法可能导致内存泄漏，我们可以用一个内置的嵌套类来专门负责释放空间：

```c++
class FileSystem {
public:
    static FileSystem& instance() {
        // 又称惰性初始化
        if (instance_ == NULL)
            instance_ = new FileSystem(); // 两个线程可能先后进入这里实例化两次
        return *instance_;
    }

private:
    FileSystem() {}
    static FileSystem* instance_;

    class Deletor {
    public:
        ~Deletor() {
            if(Singleton::instance_ != NULL)
                delete Singleton::instance_;
        }
    };
    static Deletor deletor; // 静态实例
};
```

这样一来，程序结束时会自动调用静态成员`deletor`的析构函数，删除单例类中的唯一实例。但是这种写法并不线程安全，可能会出现线程重入，我们可以加锁改进，但是有更优的写法。

饿汉版（Eager Singleton），一开始就初始化：

```c++
class Singleton {
private:
    static Singleton instance;
    Singleton();
public:
    static Singleton& getInstance() {
        return instance;
    }
}

// initialize defaultly
Singleton Singleton::instance; // 外部初始化
```

由于在`main`函数之前就就初始化，并没有线程的安全问题，但是这样会有另一个问题，编译器在初始化外部静态变量的时候的顺序是未知的，这可能影响类成员的依赖关系，因此这也不是最好的写法。

优雅的懒汉版：

```c++
class FileSystem{
public:
    static FileSystem& instance(){ // 注意这里返回值类型是引用
        static FileSystem *instance = new FileSystem(); // static保证了不会重复初始化
        return *instance;
    }

private:
    FileSystem() {}
};
```

这个写法用局部静态变量解决了上面的问题，哪怕是在多线程情况下，C++11标准也保证了本地**静态**变量只会初始化一次，因此，假设你有一个现代C++编译器，这段代码是线程安全的。

单例模式的优势有：

1. 单例只在第一次被请求时初始化，如果没有人用就不会创建实例

2. 它在运行时才实例化，也就是惰性初始化

3. 可继承

单例模式的缺点：惰性初始化会让我们丧失一部分控制权，假如某个音频只能在游戏惰性初始化，而它会因此占用某些内存带来音效的一些延迟甚至画面的掉帧，这些事情如果发生在游戏的高潮部分，就会影响游戏体验。

### 2.享元模式

### 3.观察者模式

### 4.工厂模式

### 5.策略模式

### 6.状态模式

### 7.组建模式

参考资料：[游戏设计模式](https://gpp.tkchu.me/singleton.html)及其他