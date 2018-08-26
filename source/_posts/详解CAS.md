---
title: 详解CAS
categories:
  - Java并发编程
tags:
  - Java并发编程
date: 2018-07-10 21:47:30
---

我们知道Java语言提供了synchronized来实现原子性，但是synchronized性能比较差；<!-- more -->另外Java还可以通过循环CAS(Compare And Swap，即比较并交换)来实现原子性，J.U.C中AQS同步组件、Atomic原子类操作等都是基于CAS实现的，可以说CAS是J.U.C的基石。

## 什么是CAS

CAS有3个操作数，内存值V，旧的预期值A，要修改的新值B。当且仅当预期值A和内存值V相同时，将内存值V修改为B，否则什么都不做。


## CAS在Java中的实现

CAS在J.U.C中广泛使用，特别是Atomic原子类，下面我们就用AtomicInteger作为例子讲解一下CAS在Java中的应用。


我们先来看下AtomicInteger类中定义的三个重要属性：

```
    private static final Unsafe unsafe = Unsafe.getUnsafe();
    private static final long valueOffset;

    static {
        try {
            valueOffset = unsafe.objectFieldOffset
                (AtomicInteger.class.getDeclaredField("value"));
        } catch (Exception ex) { throw new Error(ex); }
    }

    private volatile int value;
```

- ```Unsafe unsafe```  是CAS的一个核心类，封装了很多native方法，提供硬件级别的原子操作，下面我们会看到里面具体的方法。
- ```int value```  AtomicInteger类的值
- ```long valueOffset```  value在内存中的地址


有了这三个属性后我们来具体看一个方法，

```
    public final int getAndIncrement() {
        return unsafe.getAndAddInt(this, valueOffset, 1);
    }
```

这个方法提供了原子的自增操作，本身没有做太多，而是直接调用unsafe的方法，

```
    public final int getAndAddInt(Object var1, long var2, int var4) {
        int var5;
        do {
            var5 = this.getIntVolatile(var1, var2);
        } while(!this.compareAndSwapInt(var1, var2, var5, var5 + var4));

        return var5;
    }
```

在这个方法里面var1表示被操作的对象，var2表示对象某个属性在内存中的地址，var4表示对象属性增加的值，这样我们就知道```unsafe.getAndAddInt(this, valueOffset, 1)```表示的是把AtomicInteger对象的value值加1。

我们再来看下```getAndAddInt```内部具体的实现，首先是原子性获得对象某个属性的值，然后通过CAS来更新属性的值，如果有其他线程竞争导致CAS失败，则重复上面的过程直到成功为止。这里面最重要的就是调用了```compareAndSwapInt```方法。

```
    public final native boolean compareAndSwapInt(Object var1, long var2, int var4, int var5);
```

```compareAndSwapInt```方法的四个参数我们一定要理解清楚，var1表示被操作的对象，var2表示对象某个属性在内存中的地址，var4表示旧的预期值，var5表示要修改的新值。

## CAS在CPU中的实现

从上面我们知道Java实现CAS主要靠Unsafe类，Unsafe类提供的native方法最终还是要靠CPU提供的原子操作来实现，CPU提供了总线锁定和缓存锁定两个机制来保证复杂内存操作的原子性。

### 总线锁定

所谓总线锁定就是某个处理器在总线上输出一个LOCK#信号，其他处理器的请求就会被阻塞住，那么该处理器就会独占共享内存，从而达到原子性操作的目的。

总线锁定会把处理器和内存之间的通信锁住，即使不同处理器处理的是不同的数据，所以总线锁定开销比较大。

### 缓存锁定

缓存锁定就是缓存在内存区域的数据如果在加锁期间，当它执行锁操作写回内存时，处理器不在输出LOCK#信号，而是修改内部的内存地址，利用缓存一致性协议来保证原子性。缓存一致性机制可以保证同一个内存区域的数据仅能被一个处理器修改。

## CAS的优势

相比于synchronized，CAS通常都能更高效地解决原子操作，在JDK1.6之后synchronized的优化中，也大量使用了CAS的技术来提高性能。


## CAS的劣势

### 循环时间长开销大

通常CAS在循环若干次后都会取得成功，但是如果自旋CAS长时间不成功的话反而会增加系统开销。

**解决办法**：

- 可以通过控制自旋次数来解决这个问题，如果自旋次数已经超过了规定的次数则停止自旋，把线程阻塞住。

### 只能保证一个共享变量原子操作

CAS只能保证一个共享变量原子操作，对于多个共享变量，CAS无法保证操作的原子性。

**解决办法**：

- 使用锁
- 把多个共享变量放到一个类中，然后使用JDK提供的```AtomicReference```类来实现原子性


### ABA问题

CAS更新时会检查内存值有没有变化，如果没变化则更新，变化了则不做操作。但是如果内存值由A变成B，然后再由B变成A，那么CAS检查时会发现内存值并没有变化，其实已经变化了，这就是ABA问题。

**解决办法**：

- 使用版本号，如果我们每次操作时都增加一个自增的版本号，那么A -> B -> A就会变成1A -> 2B -> 3A。JDK提供了```AtomicStampedReference```类来解决ABA问题。




