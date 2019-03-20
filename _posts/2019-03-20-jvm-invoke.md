---
layout:     post
title:      深入Java虚拟机(三)
subtitle:   "Java方法调用"
date:       2019-03-20
author:     Flint
header-img: img/jvm-invoke.jpg
catalog: true
tags:
    - JVM

---

> 参考：
> https://time.geekbang.org/column/article/11539
> https://time.geekbang.org/column/article/12098

# 方法调用

##  Java字节码中的调用指令

- invokestatic：用于调用静态方法
- invokespecial： 用于调用私有实例方法、构造器；以及super关键字调用父类的实例方法和构造器；以及调用实现接口的默认方法
- invokevirtual：用于调用非私有的实例方法
- invokeinterface：用于调用接口方法
- invokedynamic： 调用动态方法

##  虚方法调用

Java中所有的非私有实例方法都会调用都会被编译成invokevirtual指令，而接口方法的调用会被编译成invokeinterface指令。这两种指令都属于JVM中的虚方法调用。

## 静态绑定

- 编译成invokestatic指令的静态方法
- 编译成invokespecial指令的方法
- 如果虚方法调用指向一个final修饰的方法，那么JVM可以静态绑定该虚方法调用的目标方法

## 动态绑定

在绝大多数情况下，JVM需要根据调用者的动态类型，来确定虚方法调用的目标方法。这就是所谓的动态绑定。
Java采用了空间换时间的策略来实现动态绑定。就是为每个类生成一张**方法表**，用于快速查找目标方法。

## 方法表

方法表的本质是一个数组，每个数组元素指向一个当前类或其祖先类的非私有实例方法。这些方法可以是具体可执行的，也可以是没有相应字节码的抽象方法。

- 子类方法表中包含父类方法表的所有方法
- 子类方法在方法表中的索引值，与它重写的父类方法索引值相同

例如：

```java
/**
* 定义一个动物的抽象父类
*/
public abstract class Animal {
    abstract void eat();
    
    public String toString() {
        return "annimal";
    }
}
```

```java
/**
* 定义一个Cat类继承Animal类
*/
public class Cat extends Animal {
    void eat() {
        System.out.println("cat eat");
    }
}
```

```java
/**
* 定义一个Dog类继承Animal类
*/
public class Dog extends Animal {
    void eat() {
        System.out.println("dog eat");
    }
    
    void run() {
        System.out.println("dog run")
    }
}
```

**Animal方法表**

| 0    | Animal.toString() |
|------|-----------------------|
| 1    | Animal.eat()      |

**Cat方法表**

| 0    | Animal.toString() |
| ---- | ----------------- |
| 1    | Cat.eat()         |

**Dog**方法表

| 0    | Animal.toString() |
| ---- | ----------------- |
| 1    | Dog.eat()         |
| 2    | Dog.run()         |

> 类加载的准备阶段，除了为静态字段分配内存，还会构造与该类相关联的方法表

## 内联缓存(inline cache)

一种加快动态绑定的优化技术，它能够缓存虚方法调用中调用者的动态类型以及该类型对应的目标方法

- 单态：指的是仅有一种状态的情况(JVM为了节省内存空间，采用的就是这种)
- 多态：有限数量下的多种状态情况
- 超多态：更多种状态的情况

