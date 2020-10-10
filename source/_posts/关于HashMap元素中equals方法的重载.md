---
title: 关于HashMap元素中equals方法的重载
date: 2020-08-31 22:58:47
categories: Java
tags: [Q&A, java]
---
在使用`HashMap`的时候，尽管重载了`equals`和`hashCode`方法，遵循了重载`equals`就必须重载`hashCode`的原则，仍然无法正确比较两个对象是否相等：
<!-- more -->
```java
import java.util.HashMap;
class test{
    public static void main(String[] args) {
        HashMap<MyClass,Integer> hm = new HashMap<MyClass,Integer>();
        MyClass o1 = new MyClass();
        MyClass o2 = new MyClass();
        hm.put(o1,1); 
        hm.get(o2);//output:null
        o1.equals(o2);//output:true
    }
}

class MyClass{
    public boolean equals(MyClass obj) {
        System.out.println("hey"); // 只被调用了一次！
        return true;
    }

    public int hashCode() {
        return 100;
    }

}
```

这是因为这里的`boolean equals(MyClass obj)`并没有覆盖掉`Object`类中的方法`boolean equals(Object obj)`，而只是它的一个拥有不同参数类型的重载版本。而`HashMap`操作的是泛型，它只会调用一般化的`Object`类中的`equals`方法。	

既然重载`equals`必须使用`Object`作为参数，就面临比较参数对象与调用对象类型的问题。一种方案是使用`getClass`方法，它适用于子类拥有不同于超类的相等语义的情形，比如`Manager`作为`Employee`的子类，必须要所有field都相等，包括`Manager`新加入的field。这样一来，`Manager`对象必须与另一个`Manager`对象相等，而不能是`Employee`的其他子类，使用`getClass`达到了这一目的。

另一方案是使用`instanceof`，它适用于所有子类都拥有同样的`equals`语义的情形。比如两个`Employee`相等的条件是他们的ID相等。这样一来，`Employee`对象可以与一个`Manager`对象相等，可以用`instanceof Employee`囊括这种情形。同时，还需要把`Employee`类中的`equals`方法声明为`final`.