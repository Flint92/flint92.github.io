---
layout:     post
title:      深入Java虚拟机(四)
subtitle:   "invokedynamic"
date:       2019-03-21
author:     Flint
header-img: img/jvm-invoke-dynamic.jpg
catalog: true
tags:
    - JVM

---

> 参考：
>
> 1. https://time.geekbang.org/column/article/12564
> 2. https://time.geekbang.org/column/article/12574
> 3. https://blog.csdn.net/aesop_wubo/article/details/48858931



# 方法句柄(MethodHandle)

> 方法句柄是一个强类型的，能够直接被执行的引用。该引用可以指向常规的静态方法或者实例方法，也可以指向构造器与字段。

MethodHandle是Java7中引入的，他类似反射中的Method，但是更灵活以及轻量。主要使用如下:

- 获取java.lang.invoke.MethodHandles.Lookup
- 创建java.lang.invoke.MethodType对象，指定方法的签名
- 通过Lookup.findXXX()方法查找指定MethodType的MethodHandle
- 调用MethodHandle.invoke()或者MethodHandle.invokeExact()方法执行方法

## 获取Lookup

```java
public class Foo {

    private static void bar(Object obj) {
        System.out.println("static bar method.");
    }

    /**
    * 获取Lookup
    */
    public static MethodHandles.Lookup lookup() {
        return MethodHandles.lookup();
    }
}
```

```java
// lookup
MethodHandles.Lookup lookup = Foo.lookup();
```

> 方法句柄的访问权限不取决于方法句柄的创建位置，而是取决于Lookup对象的创建位置。
>

## 创建MethodType

方法句柄类型(MethodType)是由方法的返回类型以及参数类型共同决定的。他是用来确定方法句柄是否适配的唯一关键。

```java
// 创建一个返回类型为void,参数类型为Object的MethodType
// 它可以对应上面Foo类的bar()方法
MethodType methodType = MethodType.methodType(void.class, Object.class);
```

## 创建MethodHandle

```java
// Lookup配合反射的Method得到MethodHandle
// method
Method method = Foo.class.getDeclaredMethod("bar", Object.class);
// 获取MethodHandle的第一种方式
MethodHandle m0 = lookup.unreflect(method);

// 获取MethodHandle的第二种方式(推荐)
MethodHandle m1 = lookup.findStatic(Foo.class, "bar", methodType);
```

## invoke/invokeExact

```java
// static bar method.
m1.invoke(new Object());
// static bar method.
m1.invokeExact(new Object());
// throw WrongMethodTypeException
m1.invokeExact("123");
```

> 调用invokeExact会严格匹配参数类型。例如，一个方法句柄的参数是类型是Object，如果你直接传入String类型，会抛出java.lang.invoke.WrongMethodTypeException异常。需要显示的将String类型转为Object类型



## 方法句柄操作

方法句柄还支持增删改参数的操作，这些操作都是通过生成另一个方法句柄来实现。

- 改：MethodHandle.asType()，invoke的自动适配类型就是利用了这个实现的。
- 删：MethodHandles.dropArguments()
- 增：它会往传入的参数中插入额外的参数，再调用另一个方法句柄，对应的API是MethodHandle.bindTo()。这个可以用来实现方法的Curry化

```java
MethodHandle m2 = m1.asType(MethodType.methodType(void.class, String.class));
// static bar method.
m2.invokeExact("123");
```

```java
MethodHandle m3 = MethodHandles.dropArguments(m1, 0,String.class);
// static bar method.
m3.invokeExact("123", new Object());
```

```java
MethodHandle m4 = m1.bindTo(new Object());
// static bar method.
m4.invokeExact();
```

# invokedynamic

> invokedynamic是Java7引入用来支持方法动态调用的。具体来说，它将调用点(Call Site)抽象成一个Java类，并且将原本由JVM控制的方法调用和方法链接暴露给应用程序。在运行过程中，每一条invokedynamic指令，将捆绑一个调用点，并且会调用该调用点链接的方法句柄。
>
> 在第一次执行invokedynamic指令时，JVM会调用该指令的Bootstrap Method，来生成调用点，并与该invokedynamic指令绑定。之后会知道调用绑定的调用点链接的方法句柄。

## lambda表达式

lambda表达式到函数式接口的转换是通过invokedynamic指令实现的。该invokedynamic指令对应的启动方法将通过ASM生成一个适配器类。

- 对于没有捕获其他变量的lambda表示式， 该invokedynamic始终返回同一个适配器类的实例
- 对于捕获其他变量的lambda表达式，每次执行invokedynamic指令都会新建一个新的适配器类的实例



