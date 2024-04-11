---
layout:     post
title:      深入Java虚拟机(五)
subtitle:   "Java内存占用"
date:       2019-03-21
author:     Flint
header-img: img/jvm-memory.jpg
catalog: true
tags:
    - JVM

---

> 参考：
>
> 1. https://time.geekbang.org/column/article/13081
> 2. https://www.cnkirito.moe/cache-line/



# Java创建对象方式

- new：通过调用构造器来创建对象，会初始化实例字段
- reflect：反射也是调用的构造器来创建对象，会初始化实例字段
- clone：通过直接复制已有对象
- 反序列化：将二进制流或者文本反序列化成对象
- Unsafe.allocateInstance：不会初始化实例字段

> 通过new出来的对象，它的内存会覆盖所有的父类的实例字段。也就是说，虽然子类无法访问父类的私有字段，
> 或者子类的实例字段隐藏了父类的同名实例字段，但是子类的实例还是会为这些父类实例字段分配内存。


# 压缩指针

> 每个Java对象都有一个对象头(object header)，这个是由标记字段和类型指针构成。
>
> 标记字段：用来存储Java对象运行时数据，如哈希码、GC信息以及锁信息
>
> 类型指针：指向该对象的类
>
> 在64位虚拟机中，对象头的标记字段占64位，类型指针也占了64位。也就是说一个对象头需要额外的16字节的开销。

由于JVM的内存是有限的，为了尽可能的减少对象的内存使用量，64位虚拟机提出来**压缩指针**，也就是将64位的对象指针压缩为32位。
压缩指针可以作用于**对象头的类型指针、引用类型字段以及引用类型数组**。

默认情况下，JVM的32为压缩指针可寻址到2的35次方个字节，也就是说32G的内存空间。当超过32G时候，压缩指针就会关闭。

> 压缩指针对应的JVM参数：-XX:+UseCompressedOops。默认开启


# 内存对齐

为了方便压缩指针寻址，需要对象的起始地址是偶数。默认情况下，JVM堆中的对象的起始地址需要对齐至8的倍数。
如果一个对象用不到8N字节，则空白的部分的空间就被浪费掉，这就是对象的填充(padding)。

当然，就算关闭了压缩指针，JVM还是会进行内存对齐。内存对齐不仅存在于对象与对象之间，还存在对象的字段之间，
例如JVM要求long、double类型的字段，以及非压缩指针状态下的引用类型的字段的地址必须是8的倍数。

内存对齐的另一个重要原因是是让字段只出现在同一CPU的缓存行。

> 内存对齐的JVM参数：-XX+ObjectAlignmentInBytes。默认为8


# 字段重排列

> JVM为了内存对齐的目的，重新分配字段的排列顺序。
>
> JVM参数：-XX:+FieldsAllocationStyle。默认1

JVM有三种重排列方式(默认1)，但是都遵循下面两个原则

- 如果一个字段占C个字节，那么它在内存中的地址偏移量也必须对齐至NC。
  这里的偏移量指的是字段的内存地址与对象的起始内存地址差值
- 子类继承字段的偏移量，必须与父类对应字段偏移量一致

```java
class A {
  long l;
  int i;
}

class B extends A {
  long l;
  int i;
}

```

```java
# 启用压缩指针时，B 类的字段分布
B object internals:
 OFFSET  SIZE   TYPE DESCRIPTION
      0     4        (object header)
      4     4        (object header)
      8     4        (object header)
     12     4    int A.i                                       0
     16     8   long A.l                                       0
     24     8   long B.l                                       0
     32     4    int B.i                                       0
     36     4        (loss due to the next object alignment)

```

> 启用压缩指针时，JVM将A类的int字段放在long前面，以填充因为long字段对齐造成的4字节缺口。
>
> 由于对象直接需要对齐8的倍数，所以最后会有4字节的填充。

```java
# 关闭压缩指针时，B 类的字段分布
B object internals:
 OFFSET  SIZE   TYPE DESCRIPTION
      0     4        (object header)
      4     4        (object header)
      8     4        (object header)
     12     4        (object header)
     16     8   long A.l
     24     4    int A.i
     28     4        (alignment/padding gap)                  
     32     8   long B.l
     40     4    int B.i
     44     4        (loss due to the next object alignment)

```

> 未启用压缩指针，B类的long类型字段的地址需要对齐8的倍数，所以中间会有4字节的填充。
>
> 最后4字节的填充也是因为对象需要对齐8的倍数。

# 伪共享

两个不同的线程分别访问一个对象的不同volatile关键字修饰的字段，逻辑上它们并没有共享的内容，因此不需要同步。
但是由于这两个字段恰好在同一个缓存行，导致的结果就是，一个字段的写入修改都会导致整个缓存行失效，也就造成了实质上的共享。

- Java可以使用字段填充的方式避免伪共享
- Java8引入了一个新的注解sun.misc.Contended用来解决伪共享