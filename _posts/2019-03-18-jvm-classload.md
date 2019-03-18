---
layout:     post
title:      深入Java虚拟机(二)
subtitle:   "Java类加载"
date:       2019-03-18
author:     Flint
header-img: img/jvm-class-load.jpg
catalog: true
tags:
    - JVM

---

> 本文参考
>
> 1. https://time.geekbang.org/column/article/11523
> 2. https://www.jianshu.com/p/5f79217f2e18

# 类的加载

> 类从被加载到虚拟机内存中开始，直到卸载出内存为止，它的整个生命周期包括了：**加载、验证、准备、解析、初始化、使用和卸载**这7个阶段。其中，**验证、准备和解析这三个部分统称为连接（linking）**。
>

![类加载](https://img-blog.csdn.net/20140317163048593)

# 加载

> 加载是指查找字节流，并且根据据此创建类的过程。对于数组来说，它是没有对应的字节流的，而是Java虚拟机直接生成的。对于其他类或者接口，JVM需要借助类加载器来完成查找字节流的过程。
>

## Java中的类加载器

![类加载器](https://upload-images.jianshu.io/upload_images/3251891-d34761b5a29e065b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/212/format/webp)

- 启动类加载器(Bootstrap ClassLoader)是由c++实现的，没有对应的Java对象表示，所以永远返回null。
- 除了Bootstrap ClassLoader之外，其他的类加载器都是java.lang.ClassLoader的子类，因此有对应的Java对象。这些类加载器都需要先由另一个类加载器，比如Bootstrap ClassLoader加载到JVM中，才能执行其他类的加载。

> 在Java9之前，Bootstrap ClassLoader主要负责加载最为基础、最为重要的类，比如存放在JRE的lib目录下jar包中的类，以及由JVM参数-Xbootclasspath指定的类。
> 扩展类加载器(Extension ClassLoader)负责加载相对重要、又通用的类，比如存放在JRE的lib/ext目录下jar包中的类，以及由系统变量java.ext.dirs指定的类。
> 应用类加载器(Application ClassLoader)负责应用程序下的类的加载。
>
> Java9之后引入了模块系统，扩展类加载器改名为平台类加载器(Platfom ClassLoader)， JavaSe除了少数关键模块由Bootstrap ClassLoader加载外，其他模块都由Platform ClassLoader加载。
> 

## 双亲委派机制
在Java中，一个类加载器尝试去加载一个类时候，它会首先委托父类加载器去加载这个类，依次传递到Bootstrap ClassLoader，如果父类加载器无法加载此类，才会尝试在子类加载器完成加载。

- 双亲委派机制可以保证每个类只会被加载一次，避免了重复加载
- 双亲委派机制保证了每个类尽可能被加载
- 双亲委派机制有效避免了某些恶意类的加载

> 在JVM中，类的唯一性是由类加载器的实例以及类的全类名共同决定的。
> 

# 链接
> 链接是指将创建的类合并到Java虚拟机中，使之能够执行的过程。分为验证、准备、解析三个过程。
> 

- 验证：确保类满足JVM约束条件
- 准备：为类的静态字段分配内存，构造跟类层次相关的数据结构。
- 解析：在class文件被加载到JVM之前，Java编译器会给类的字段和方法都生成一个符号引用。解析的目的就是将符号引用解析成为实际的引用。

# 初始化

- 静态字段被final修饰，并且类型是基本数据类型或者String类型，会被标记为常量值(ConstantValue)，初始化由JVM直接执行。
- 其他直接赋值或者通过静态代码块赋值代码都会被Java编译器放到同一个方法中<clinit>。
- 初始化就是为常量值赋值以及执行<clinit>方法。执行<clinit>方法是加锁的，确保只会被执行一次。
- 只有初始化完成之后，类才是可执行状态。
