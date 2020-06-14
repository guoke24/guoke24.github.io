---
layout:     post  
title:      "Synchronized 和 ReentrantLock"  
subtitle:   "同一时刻，只有一个！"  
date:       2020-04-27 12:00:00  
author:     "GuoHao"  
header-img: "img/guitar.jpg"  
catalog: true  
tags:  
    - Android    
    - 异步多线程 

---

## 参考

[第08讲：既生 Synchronized，何生 ReentrantLock](https://kaiwu.lagou.com/course/courseInfo.htm?courseId=67#/detail/pc?id=1862)，付费内容，仅仅做记录使用，若侵权则删!

## synchronized

synchronized 可以用来修饰以下 3 个层面：

修饰实例方法；

修饰静态类方法；

修饰代码块。

### 修饰实例方法

> 这种情况下的锁对象是当前实例对象，因此只有同一个实例对象调用此方法才会产生互斥效果，不同实例对象之间不会有互斥效果。

### 修饰静态类方法

> 如果 synchronized 修饰的是静态方法，则锁对象是当前类的 Class 对象。因此即使在不同线程中调用不同实例对象，也会有互斥效果。

### 修饰代码块

> synchronized 作用于代码块时，锁对象就是跟在后面括号中的对象。上图中可以看出任何 Object 对象都可以当作锁对象。

这种情况跟情况一类似...

### 实现细节

synchronized 既可以作用于方法，也可以作用于某一代码块。但在实现上是有区别的。

先说共同点：编译后的字节码中都会插入 **1 个 monitorenter 和 2 个 monitorexit 指令**；多出的一个 monitorexit 指令是为了异常退出。

区别在于：插入的位置不同；

**对于代码块来说，指令在代码块的前后和异常的位置；**

**对于方法来说，指令在方法的开始和结束（或异常）位置；**（被 synchronized 修饰的方法在被编译为字节码后，在方法的 flags 属性中会被标记为 ACC_SYNCHRONIZED 标志。当虚拟机访问一个被标记为 ACC_SYNCHRONIZED 的方法时，会自动在方法的开始和结束（或异常）位置添加 monitorenter 和 monitorexit 指令。）

引用 [第08讲：既生 Synchronized，何生 ReentrantLock](https://kaiwu.lagou.com/course/courseInfo.htm?courseId=67#/detail/pc?id=1862) 的图片：

> 关于 monitorenter 和 monitorexit，可以理解为一把具体的锁。在这个锁中保存着两个比较重要的属性：计数器和指针。
> - 计数器代表当前线程一共访问了几次这把锁；

> - 指针指向持有这把锁的线程。

> 用一张图表示如下：

> ![](https://s0.lgstatic.com/i/image3/M01/89/8E/Cgq2xl6X-COAEskYAABd1Qkprak432.png)


> 锁计数器默认为0，当执行monitorenter指令时，如锁计数器值为0 说明这把锁并没有被其它线程持有。那么这个线程会将计数器加1，并将锁中的指针指向自己。当执行monitorexit指令时，会将计数器减1。

## ReentrantLock

### 更细节

ReentrantLock 可以比 synchronized 做更多的事，但是相对的，操作粒度也会更细。即 ReentrantLock 需要手动的解锁！

### 公平锁和非公平锁

此外，ReentrantLock 还支持公平锁和非公平锁；通过构造函数传入 true 或 false 决定；

公平锁，就是讲究先来后到，要排队。

### 读写锁

读写锁（ReentrantReadWriteLock）如何使用：

1.创建读写锁对象：

![](https://s0.lgstatic.com/i/image3/M01/89/8E/Cgq2xl6X-CSAfAiHAAAlGuwPEXA557.png)

2.通过读写锁对象分别获取读锁（ReadLock）和写锁（WriteLock）：

![](https://s0.lgstatic.com/i/image3/M01/03/49/CgoCgV6X-CWAEmShAAApkBH8nBM233.png)

3.使用读锁（ReadLock）同步缓存的读操作，使用写锁（WriteLock）同步缓存的写操作：

![](https://s0.lgstatic.com/i/image3/M01/10/78/Ciqah16X-CWAHTM4AACOosEvECg851.png)

具体案例看原链接 [第08讲：既生 Synchronized，何生 ReentrantLock](https://kaiwu.lagou.com/course/courseInfo.htm?courseId=67#/detail/pc?id=1862) ...

## 小结

synchronized 使用更简单，加锁和释放锁都是由虚拟机自动完成，而 ReentrantLock 需要开发者手动去完成。但是很显然 ReentrantLock 的使用场景更多，公平锁还有读写锁都可以在复杂场景中发挥重要作用。

## demo

之前有写过一些同步的 demo 代码上传到了 Github，在 [该链接](https://github.com/guoke24/Anything/tree/master/app/src/main/java/com/guohao/anything/sync) 。



