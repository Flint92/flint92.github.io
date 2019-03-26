---
layout:     post
title:      深入Java虚拟机(八)
subtitle:   "synchronized实现"
date:       2019-03-26
author:     Flint
header-img: img/jvm-synchronized.jpg
catalog: true
tags:
    - JVM


---

> 
>参考：
>
>1、<https://time.geekbang.org/column/article/13530>
>
>2、<http://www.iocoder.cn/JUC/sike/synchronized/>

# Synchronized实现

当用synchronized标记方法时，编译而成的字节码将会包含monitorenter（加锁）和monitorexit（解锁）指令。一个monitorenter指令对应多个monitorexit指令，这是因为JVM要保证所获得的锁在正常执行路径以及异常执行路径上都能够被解锁。

这里monitorenter和monitorexit操作所对应的锁对象是隐式的。

- 对于实例方法，锁是当前实例对象
- 对于静态方法，锁是当前类的Class对象
- 对应同步代码块，锁是括号里面的对象

关于monitorenter和monitorexit的作用，可以抽象理解为每个锁对象拥有一个锁计数器和一个指向持有该锁的线程的指针。

当指向monitorenter时候，如果目标锁对象的计数器为0，说明它还没被其他线程所持有，在这种情况下，JVM会将锁对象的持有线程指向当前线程，并将计数器+1；如果目标锁对象的计数器不为0，如果锁对象持有的线程就是当前线程，那么JVM直接将计数器+1，否则需要等待直至持有锁的线程释放锁。

当执行monitorexit时候，JVM需要将锁对象的计数器-1，当计数器减为0时候，将锁对象持有的线程设为null，此时，说明锁被释放掉。

> 可以看出synchronized锁是一个可重入锁

# 重量级锁

重量级锁通过对象内部的Monitor实现。其中，Monitor 的**本质**是，依赖于底层操作系统的 [Mutex Lock](http://dreamrunner.org/blog/2014/06/29/qian-tan-mutex-lock/) 实现。操作系统实现线程之间的切换，需要从用户态到内核态的切换，切换成本非常高。

# 轻量级锁

对于轻量级锁，其性能提升的依据是：“**对于绝大部分的锁，在整个生命周期内都是不会存在竞争的**”。

轻量级锁采用CAS操作，将锁对象的标记字段替换为一个指针，指向当前线程栈上的一块空白内存，存储着锁原本的标记字段。

![轻量级锁](<https://gitee.com/chenssy/blog-home/raw/master/image/sijava/201812081005.png>)

# 偏向锁

**从始至终只有一个线程来请求锁**

偏向锁，只会在第一次请求时采用CAS操作，在锁对象的标记字段中记录下当前线程的地址。在此后的运行过程中，持有该偏向锁的线程的加锁操作将直接返回。

![偏向锁](<https://gitee.com/chenssy/blog-home/raw/master/image/sijava/201812081006.png>)