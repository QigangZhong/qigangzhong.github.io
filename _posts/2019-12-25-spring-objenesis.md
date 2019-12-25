---
layout: post
title:  "spring中的objenesis"
categories: spring
tags: spring
author: 网络
---

* content
{:toc}











## objenesis介绍

有个[objenesis的站点](http://objenesis.org/index.html)，专门介绍objenesis。在Java中已经支持通过Class.newInstance()动态实例化Java类，但是这需要Java类有个适当的构造器。很多时候一个Java类无法通过这种途径创建，例如：

* 构造器需要参数
* 构造器有副作用
* 构造器会抛出异常

Objenesis可以绕过上述限制。它一般用于：

* 序列化、远程处理和持久化：无需调用代码即可将Java类实例化并存储特定状态。
* 代理、AOP库和Mock对象：可以创建特定Java类的子类而无需考虑super()构造器。
* 容器框架：可以用非标准方式动态实例化Java类。例如Spring引入Objenesis后，Bean不再必须提供无参构造器了。

## 使用

在站点上有个[20s教程](http://objenesis.org/tutorial.html)，示例如何使用objenesis：

```java
bjenesis objenesis = new ObjenesisStd(); // or ObjenesisSerializer
MyThingy thingy1 = (MyThingy) objenesis.newInstance(MyThingy.class);

// or (a little bit more efficient if you need to create many objects)

Objenesis objenesis = new ObjenesisStd(); // or ObjenesisSerializer
ObjectInstantiator thingyInstantiator = objenesis.getInstantiatorOf(MyThingy.class);

MyThingy thingy2 = (MyThingy)thingyInstantiator.newInstance();
MyThingy thingy3 = (MyThingy)thingyInstantiator.newInstance();
MyThingy thingy4 = (MyThingy)thingyInstantiator.newInstance();
```

## spring中的objenesis

spring框架集成了objenesis，在spring-core包中有独立的objenesis目录

![spring-core-objenesis.jpg](/images/spring/spring-core-objenesis.jpg)

```java
public class MyClass {
    private String name;

    //定义有参构造函数，则默认的无参构造函数不再生成
    public MyClass(String name){
        this.name = name;
    }

    public void myMethod(){
        System.out.println(name);
    }
}
```

```java
import org.springframework.objenesis.Objenesis;
import org.springframework.objenesis.ObjenesisStd;
import org.springframework.objenesis.instantiator.ObjectInstantiator;

public class ObjenesisTest {
    public static void main(String[] args) throws IllegalAccessException, InstantiationException {
        //如果一个类没有无参构造函数，JDK实例化会报错
        /*MyClass myClass1 = MyClass.class.newInstance();
        myClass1.myMethod();*/

        //但是通过objenesis实例化则没有问题
        Objenesis objenesis = new ObjenesisStd();
        ObjectInstantiator objectInstantiator = objenesis.getInstantiatorOf(MyClass.class);
        MyClass myClass2 = (MyClass) objectInstantiator.newInstance();
        myClass2.myMethod();
    }
}
```

## 参考
