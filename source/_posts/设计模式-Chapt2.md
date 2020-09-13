---
title: 设计模式-Chapt2
date: 2020-09-07 21:01:03
categories: Design_patterns
tags: [notes, design pattern]
---
### 文档结构

所有对象对应一个抽象基类`Glyph`（图元），图形元素（字符和图像）以及结构元素（行和列）都是它的子类。这样的设计做到了对文本和图形的一致对待，也没有过分强调单个元素和元素组之间的差别。	

| 责任     | 操作                                  |
| -------- | ------------------------------------- |
| 表现     | virtual void Draw(Window*)            |
| 表现     | virtual void Bounds(Rect&)            |
| 点击检测 | virtual bool Intersects(Const Point&) |
| 结构     | virtual void Insert(Glyph*, int);     |
| 结构     | virtual void Remove(Glyph*)           |
| 结构     | virtual Glyph* Child(int)             |
| 结构     | virtual Glyph* Parent()               |

`Draw`负责让图元画出自己，`Bounds`指出图元占用多大空间，`Intersects`判断一个指定的点是否与图元相交，结构操作与图元的父图元和子图元相关。

### 格式化

格式化文档的算法应该被封装起来，独立于文档结构之外。

`Composition`是`Glyph`的子类，拥有`Compositor`对象，`Compositor`封装了具体的格式化算法。

支持文档物理结构的类和支持不同格式化算法的类实现了分离。

### 修饰用户界面

如果想添加边界或滚动条，可以选择创建`Composition`的子类`BorderedComposition`和`ScrollableComposition`，但是组件多了有$2^n$排列组合，显然不可能这么做下去。可以通过对象组合的方式实现。`Border`应该是一个`Glyph`的子类，因为客户不应区分图元是否有边界而使用不同的接口。于是可以引入一个抽象基类`Monoglyph`，它是只有一个子图元的图元，把所有对这个组件的请求转发给它的子图元。这样一来，`Border`就可以继承`MonoGlyph`，它先调用`MonoGlyph`的`Draw`函数，然后画出边界。

### 支持多种视感标准

如果直接用`ScrollBar* sb = new MotifScrollBar`来创建实例，相当于编译时定死了视感标准。此时就要用工厂`ScrollBar* sb = guiFactory->CreateScrollBar()`.这里`guiFactory`是`MotifFactory`的实例，每种Factory都是`GUIFactory`抽象基类的子类。

### 支持多种窗口系统

我们把不同窗口系统的公共部分抽象出来加以封装成`Window`类，而不同窗口系统的区别则封装在`WindowImp`类里。`Window`类通过调用成员类`WindowImp`的对应函数实现对应功能，这样一来，不同窗口系统接口的巨大差异就被`WindowImp`隐藏了。这正是Bridge模式的一个例子，`Window`支持窗口的逻辑概念，而`WindowImp`描述了窗口的不同实现，两个独立演化的，分离的类层次一起工作。

### 用户操作

定义一个`Command`抽象类，它的不同子类按照各自的需求实现`execute`方法，所有的用户操作（如点击菜单项），只是执行了这个操作的对象的`Command->Execute()`方法。同时，维护一个最近执行的`Command`列表就能方便地实现Undo和Redo操作。

### 拼写检查和段字处理

#### 1.遍历

我们需要访问子图元中的对象来分析文本，这就需要了解图元存储子图元的内部数据结构。为了保持封装，引入迭代器类`Iterator`.每种内部数据结构（数组、链表）都对应一个`Iterator`的子类。图元使用`CreateIterator`方法创建与其数据结构对应的迭代器，这就保证了遍历算法对用户透明。

#### 2.分析

显然，分析工作不能在迭代器类系列中进行，因为我们希望降低耦合。如果把分析工作放在图元类里，似乎是可以的，因为对不同的图元类型，分析工作显然不同。比如，拼写检查应该考虑字符图元，颜色分割应该考虑可见的图元。但是这样一来，每增加一种新的分析，就要改变每一个图元类，并且会导致`Glyph`的接口越来越大，模糊它的基本接口。

因此，应该用独立对象封装分析方法。如果我们对某种分析（拼写分析）定义一个类`SpellingChecker`，那就可以给图元定义一个接口`void CheckMe(SpellingChecker&)`，遍历图元时使用这个接口即可做到拼写检查。但是这样要为每种分析增加一个`Checker`，更严重的是，要为图元类增加一个对应的`CheckMe`接口。

于是引入`Visitor`类，它对不同类型的`Glyph`子类有一个`VisitXXX`函数。`CheckMe`也改为一个更加通用的名字`Accept(Visitor&)`.添加一个新分析类型，只需要建立一个新的`Visitor`子类。不过一旦图元类有了一个新的子类，就需要对所有的`Visitor`添加一个对应的`VisitXXX`函数，对于文字处理应用，增加一个图元不是一个合理需求，所以这样抽象满足我们的需要。

