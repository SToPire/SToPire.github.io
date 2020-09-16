---
title: 设计模式-Chapt3
date: 2020-09-16 21:19:01
categories: Design_patterns
tags: [notes, design pattern]
---

创建型模式可以分为两类，一种是生成创建对象的类的子类，对应`Factory Method`模式，另一种方式更多依赖于对象组合，即创建一个“工厂对象”，专门负责创建产品对象，对应`Abstract Factory`,`Builder`和`Prototype`模式。
<!-- more -->
## Abstract Factory

提供一个接口以创建一系列相关或相互依赖的对象，而无须指定它们具体的类。

适用于：一个系统要独立于它的产品的创建、组合和表示；一个系统要由多个产品系列中的一个来配置。

![](/images/DesignPattern_3/1.png)

| 对象            | 功能                                                 |
| --------------- | ---------------------------------------------------- |
| AbstractFactory | 声明一个创建抽象产品对象的操作接口                   |
| ConcreteFactory | 实现创建具体产品对象的操作                           |
| AbstractProduct | 为一类产品对象声明一个接口                           |
| ConcreteProduct | 定义一个将被相应的具体工厂创建的产品对象             |
| Client          | 仅使用由AbstractFactory和AbstractProduct类声明的接口 |

在运行时创建一个`ConcreteFactory`实例，创建具体类型的产品对象。`AbstractFactory`将产品对象的创建延迟到它的`ConcreteFactory`子类。

客户不接触到具体的产品类名，同时也易于改变产品序列，有利于产品的一致性。

然而若是要增加一个新种类的产品（即ProductC)，就必须改变`AbstractFactory`的接口，一种改进措施是只实现一个`Make`接口，然后用一个标识符作为参数指明创建的是何种对象。这样做的问题是返回值会是一个相同的抽象接口，用户必须依赖强制类型转换获得具体的产品类别。

## Builder

将一个复杂对象的构建和它的表示分离，使同样的构建过程可以创建不同的表示。

适用于：创建复杂对象的算法应该独立于该对象的组成部分以及它们的装配方式时；构造过程必须允许被构造的对象有不同的表示时。

![](/images/DesignPattern_3/2.png)

| 对象            | 功能                                                         |
| --------------- | ------------------------------------------------------------ |
| Builder         | 为“创建产品的各部分”提供抽象接口                             |
| ConcreteBuilder | 实现Builder接口，构造装配产品；定义跟踪创建的产品并提供检索接口(GetResult) |
| Director        | 构造一个使用Builder接口的对象                                |
| Product         | 表示被构造的复杂对象；它包含了定义其组成部件的类             |

用户创建`Director`对象并配置`Builder`，一旦导向器生成产品部件就通知生成器构造产品，用户通过生成器检索产品。

`Builder`隐藏了产品的内部表示和装配过程；分离了构造和表示过程，比如可以对RTF, SGML格式的文档各自使用一个阅读器(`Director`)，它们可以使用同样的`Builder`（如ASCIIText）把该格式的文档转为ASCII字符。

`Builder`不需要把每个构件的生成方法定义为纯虚函数，用户可以按需重载而非全部重载。

## Factory Method

定义一个用于创建对象的接口，让子类决定实例化哪一个类。(Virtual constructor)

适用于：一个类不知道它必须创建的对象的类时；一个类希望它的子类来指定它所创建的对象的时候

![](/images/DesignPattern_3/3.png)

| 对象            | 功能                                    |
| --------------- | --------------------------------------- |
| Product         | Factory Method所创建对象的接口          |
| ConcreteProduct | 实现Product接口                         |
| Creator         | 调用工厂方法以创建一个Product对象       |
| ConcreteCreator | 重定义工厂方法，返回ConcreteProduct实例 |

代码只处理`Product`接口，`Factory Method`使用户不再需要将与应用相关的具体类名包含在代码中。

c++的`template`就是一种工厂方法，而且免去了创建`ConcreteCreator`子类的过程，直接得到`ConcreteCreator`.

### 与`Abstract Factory`的区别

参考资料：https://stackoverflow.com/questions/5739611/what-are-the-differences-between-abstract-factory-and-factory-design-patterns

`Factory Method`是方法，`Abstract Factory`是一个对象，一般由很多个`Factory Method`组成。

工厂方法用于创建某一特定产品，抽象工厂用于创建一系列相关的产品。

在使用抽象工厂模式时，一个类会把创建其他对象的职责委托给工厂对象；而使用工厂方法模式时，`Creator`会依赖继承序列里的`ConcreteCreator`来完成创建工作。

`Factory Method`:

```c++
class A {
    public void doSomething() {
        Foo f = makeFoo();
        f.whatever();   
    }

    protected Foo makeFoo() {
        return new RegularFoo();
    }
}

class B extends A {
    protected Foo makeFoo() {
        //subclass is overriding the factory method 
        //to return something different
        return new SpecialFoo();
    }
}
```

`Abstract Factory`:

```c++
class A {
    private Factory factory;

    public A(Factory factory) {
        this.factory = factory;
    }

    public void doSomething() {
        //The concrete class of "f" depends on the concrete class
        //of the factory passed into the constructor. If you provide a
        //different factory, you get a different Foo object.
        Foo f = factory.makeFoo();
        f.whatever();
    }
}

interface Factory {
    Foo makeFoo();
    Bar makeBar();
    Aycufcn makeAmbiguousYetCommonlyUsedFakeClassName();
}

//need to make concrete factories that implement the "Factory" interface here
```

## Prototype

用原型实例指定创建对象的种类，并通过拷贝这些原型创建新的对象。

适用于：一个系统应该独立于它的产品创建、构成和表示时；当要实例化的类是在运行时指定时；避免创建一个与产品类层次平行的工厂类层次时；一个类的实例只有几种可能的状态组合，可以用原型复制比每次手工填充状态方便

![](/images/DesignPattern_3/4.png)

| 对象              | 功能                                 |
| ----------------- | ------------------------------------ |
| Prototype         | 声明一个克隆自身的接口               |
| ConcretePrototype | 实现一个克隆自身的操作               |
| Client            | 让一个原型克隆自身以创建一个新的对象 |

`Prototype`模式允许用户在运行时建立/删除原型，以将一个新的具体产品类并入系统。同时，不需要像`Factory Method`那样产生一个与产品类层次平行的`Creator`层次。

实现的时候，一般会使用一个原型管理器保存可用的原型，实现原型的动态创建和销毁。

## Singleton

保证一个类仅有一个实例，提供一个访问它的全局访问点。

有些类应该保持唯一性，可以让这个类自身负责保存它的唯一实例，并提供一个访问该实例的方法。

![](/images/DesignPattern_3/5.png)

| 对象      | 功能                                               |
| --------- | -------------------------------------------------- |
| Singleton | 定义一个`Instance`操作，使用户可以访问它的唯一实例 |

`Singleton`模式可以有子类，并且用这个子类的唯一实例来初始化`Instance`，实现了操作的精化。

可以通过将类的构造函数设为`protected`做到这一点，相比于把`Singleton`定义为全局或者静态变量，这样做保证了只有一个实例会被创建。

