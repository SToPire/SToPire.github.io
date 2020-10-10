---
title: lambda表达式，匿名内部类，代理
date: 2020-10-10 14:33:54
categories: Java
tags: [notes, Java]
---
参考资料：https://www.zhihu.com/question/20794107/answer/658139129

<!-- more -->

## lambda表达式

### 语法

lambda表达式由3部分组成：

| 参数列表         | 箭头运算符 | 函数体 |
| ---------------- | ---------- | ------ |
| `(int x, int y)` | `->`       | `x+y`  |

函数体可以是一个代码块。如果可以推断出参数类型，可以省略参数列表中的类型；在此基础上如果只有一个参数，小括号也可以省略。

lambda表达式的返回类型总是自动推导得出。

lambda表达式拥有与它嵌套其中的块一样的作用域，尽管lambda表达式可能会拥有一个大括号包围的代码块。

因此，lambda可以使用外围作用域中变量的值，称为捕获值。捕获的变量初始化后不能再被赋新值，称为effectively final。

### 函数式接口

函数式接口：抽象方法数目为1的接口。（接口中`default`修饰的方法，以及重新声明的`Object`类方法都不属于抽象方法）

在使用函数式接口的地方，可以用lambda表达式替换，这也是lambda唯一能做的事情。例：

```java
Arrays.sort(words, (first,second) -> first.length - second.length);
```

一些函数式接口：

| 函数式接口     | 参数类型 | 返回类型  | 抽象方法名 | 描述                         |
| -------------- | -------- | --------- | ---------- | ---------------------------- |
| `Runnable`     | 无       | `void`    | `run`      | 作为无参数或返回值的动作运行 |
| `Supplier<T>`  | 无       | `T`       | `get`      | 提供一个T类型的值            |
| `Consumer<T>`  | `T`      | 无        | `accept`   | 返回一个T类型的值            |
| `Predicate<T>` | `T`      | `boolean` | `test`     | 谓词，返回布尔值             |

此外，对于primitive类型还有这些接口的基本类型版本。

### 方法引用

如果想要传递的lambda表达式只是调用了一个现成的方法而已（也就是这个方法已经可以完成目的了），可以使用方法引用的形式直接传递这个方法，形式是用`::`分割对象/类名与方法名，比如：

```java
Arrays.sort(strings, String::compareToIgnoreCase);
```

相当于

```
Arrays.sort(strings, (x, y) -> x.compareToIgnoreCase(y));
```

传递的方法有3种情形：

| 类型                     | 例子                         | 相当于                             |
| ------------------------ | ---------------------------- | ---------------------------------- |
| `object::instanceMethod` | `System.out::println`        | `x->System.out.println(x)`         |
| `Class::staticMethod`    | `Math::pow`                  | `(x, y) -> Math.pow(x, y)`         |
| `Class::instanceMethod`  | `String:compateToIgnoreCase` | `(x, y)->x.compareToIgnoreCase(y)` |

使用`Class:new`可以调用对应的类构造器，使用哪一个构造器会从上下文推断出来。

## 匿名内部类

如果只希望创建某个类的一个对象，就不需要给这个类起名了。

```j
new SuperType(construction parameters)
	{
		inner class methods and data
	}
```

创建的匿名类是`SuperType`的子类。匿名类没有名字，自然也没有构造函数，因此构造器参数被传给了超类的构造器。例如：

```java
Date mydate = new Date()
	{
		public int compareTo(Date other){
			balabala;
			return super.compareTo(other);
		}
	}
```

`mydate`指向一个`Date`的子类对象，它重载了`compareTo`方法。

### tip1

如果需要向方法传递一个以后不会再使用的对象作为参数，而没有一个合适的构造函数来构建这个对象，可以借助匿名内部类的语法，称为双括号初始化(double brace initialization).

```java
/* void foo(ArrayList<String> arr)*/
foo(new ArrayList<String>() {{ add("Alice"); add("Bob"); }});
```

第一个大括号建立了`ArrayList`的匿名子类，第二个大括号是object initialization block.

### tip2

调试时如何在静态方法中获得当前类名呢？使用`getClass()`是不行的，因为静态方法没有获得`this`指针作为参数。曲线救国：

```java
System.out.println("Something awful happened in " + new Object(){}.getClass().getEnclosingClass());
```

我们创建了一个`Object`类的匿名子类的匿名对象并寻找它的外围类的名称，这个外围类自然是定义匿名子类的类，也就是静态方法所在的类。

## 代理

如果需要在现有类（目标类）的所有方法中加入新的操作（比如，打印日志），静态代理的做法是编写一个代理类，它应该与现有类拥有完全相同的接口。代理对象has a目标对象，并在打印日志之后调用目标对象的对应方法。但是，这样做需要为每个现有的类创建一个对应的代理类。

我们知道，代理对象与目标对象有着相同的接口。使用动态代理，可以直接根据接口所对应的class对象创造代理对象，跳过了编写代理类的步骤。

为什么不能直接用接口创建代理对象？接口是抽象类。

```java
Class ProxyClass = Proxy.getProxyClass(Adder.class.getClassLoader(), Adder.class);
Constructor ProxyClassConstructor = ProxyClass.getConstructor(InvocationHandler.class);
Adder ProxyAdder = (Adder) ProxyClassConstructor.newInstance(new InvocationHandler() {
    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Exception {
        System.out.println("Proxy");
        AdderImpl _myAdderImpl = new AdderImpl();
        Object result = method.invoke(_myAdderImpl, args);
        return result;
    }
});
```

使用`getProxyClass`方法，获得了实现了接口`Adder`的代理类所对应的`Class`对象（并没有真的编写这个代理类，只是通过反射获得`Class`对象）。使用代理类接受`InvocationHandler`参数的构造函数创建了一个`Adder`对象，这就是实现了`Adder`接口的代理对象。调用这个`ProxyAdder`对象的方法，如`add(int, int)`，都会被导向作为参数的`InvocationHandler`中重载的`invoke`方法。我们需要在这个方法中创建目标对象，完成实际操作，打印日志等附加操作也在这个方法中完成。因此动态代理的结构是：代理对象中方法->`InvocationHandler`中`invoke`方法->目标对象的方法。

以上过程改为非硬编码的形式如下：

```java
public class test {
    public static void main(String[] args) throws Exception {
        AdderImpl myAdderImpl = new AdderImpl();
        Adder myAdderProxyImpl = (Adder) getProxyObject(myAdderImpl);
        System.out.println(myAdderProxyImpl.add(1, 2));
    }

    //从目标对象得到代理对象
    public static Object getProxyObject(Object target) throws Exception {
        Class proxyClass = Proxy.getProxyClass(target.getClass().getClassLoader(), target.getClass().getInterfaces());
        Constructor proxyClassConstructor = proxyClass.getConstructor(InvocationHandler.class);
        return proxyClassConstructor.newInstance(new InvocationHandler() {
            @Override
            public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
                System.out.println("Proxy");
                return method.invoke(target, args);
            };
        });
    }
}
```

事实上，`getProxyClass`方法已经为deprecated.替代方法是`newProxyInstance(ClassLoader loader, Class<?>[] interfaces, InvocationHandler h)`，得到代理类所对应的`Class`对象这一步也可以略去。

```java
public class test {
    public static void main(String[] args) throws Exception {
        AdderImpl myAdderImpl = new AdderImpl();
        Adder myAdderProxyImpl = (Adder) getProxyObject(myAdderImpl);
        System.out.println(myAdderProxyImpl.add(1, 2));
    }

    public static Object getProxyObject(Object target) throws Exception {
        return Proxy.newProxyInstance(
            target.getClass().getClassLoader(),
            target.getClass().getInterfaces(), 
            new InvocationHandler(){
                @Override
                public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
                    System.out.println("invoking!");
                    return method.invoke(target, args);
                };
            }
        );
    }
}
```

