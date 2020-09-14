---
title: 关于HashMap元素中equals方法的重载
date: 2020-08-31 22:58:47
categories: Java
tags: [Q&A, java]
---
在使用`HashMap`的时候，尽管重载了`equals`和`hashCode`方法，仍然无法正确比较两个对象是否相等：
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

