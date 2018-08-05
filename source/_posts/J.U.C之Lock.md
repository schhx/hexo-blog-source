---
title: J.U.C之Lock
categories:
  - Java并发编程
tags:
  - Java并发编程
date: 2018-07-29 14:54:35
---

锁是并发编程不可或缺的一部分，它能够控制多个线程访问共享资源的方式。<!-- more -->Java语言提供了synchronized关键字使得我们很容易的实现锁功能，synchronized实现的锁是隐式锁，在Java SE 1.5之后J.U.C提供了显示锁Lock（接口以及相关实现类），它提供了和synchronized关键字类似的同步功能，只是在使用时需要显示地获取和释放锁。

## Lock接口

Lock接口定义了锁获取和释放的基本操作：

```
public interface Lock {

    // 获取锁，获取到锁后返回，获取不到则阻塞住
    void lock();

    // 获取锁并响应中断，如果线程被中断则抛出InterruptedException
    void lockInterruptibly() throws InterruptedException;

    // 尝试非阻塞获取锁，这个方法会立即返回，如果获取到锁则返回true，否则返回false
    boolean tryLock();

    // 超时获取锁
    // 1.超时时间内获取到锁，返回true   
    // 2.等待时被中断，抛出InterruptedException   
    // 3.超时时间结束未获取到锁，返回false
    boolean tryLock(long time, TimeUnit unit) throws InterruptedException;

    // 释放锁
    void unlock();

    // 获取等待通知组件，该组件和当前锁绑定
    // 当前线程只有获取到了锁，才能调用该组件的```wait()```方法，而调用后，当前线程释放锁
    Condition newCondition();
}
```

从上面的定义可以看出Lock接口提供了synchronized不具备的一些特性，比如

- 尝试非阻塞地获取锁
- 获取锁并响应中断
- 超时获取锁

## 基本用法

不要把获取锁的过程写在try块里，因为如果在获取锁的过程中发生了异常，会导致锁无故释放。

使用lock时应该始终记得释放锁，所以应该在finally块里释放锁。

```
    Lock lock = new ReentrantLock();
    lock.lock();
    try{
        // do something
    } finally {
        lock.unlock();
    }
```

## ReentrantLock

ReentrantLock实现了Lock接口，其内部是基于AQS来实现，如果你熟悉AQS的话，那么理解ReentrantLock内部的实现就会比较容易；如果你对AQS不熟悉的话，可以看先下[详解AQS](https://schhx.github.io/2018/07/22/%E8%AF%A6%E8%A7%A3AQS/)这篇文章。

ReentrantLock里面大部分的功能都是委托给内部类Sync来实现的，Sync继承AQS并重写了独占式释放同步状态的tryRelease方法，另外Sync内还有一个重要的方法nonfairTryAcquire，它实现了非公平地获取同步状态的过程。Sync有两个子类NonfairSync和FairSync，在子类中重写了AQS的独占式获取同步状态的tryAcquire方法，从这里我们知道

- ReentrantLock是一个独占锁，同一时刻只有一个线程可以获得锁
- ReentrantLock内部实现了公平锁和非公平锁，在实例化对象时可以指定，不指定的话默认是非公平锁

```
    public ReentrantLock() {
        sync = new NonfairSync();
    }

    public ReentrantLock(boolean fair) {
        sync = fair ? new FairSync() : new NonfairSync();
    }
```

### 获取锁

ReentrantLock获取锁是委托给Sync来实现的，Sync并没有实现lock方法而是交给子类来实现。

```
    // ReentrantLock的lock方法
    public void lock() {
        sync.lock();
    }
    
    // Sync的lock方法
    abstract void lock();
```

#### 非公平锁 NonfairSync

NonfairSync实现lock方法，它首先尝试通过一次CAS快速获取同步状态，如果获取成功则设置自己独占同步状态，否则调用AQS的acquire方法。

```
    final void lock() {
        // 通过CAS获取同步状态
        if (compareAndSetState(0, 1))
            // 设置当前线程独占同步状态
            setExclusiveOwnerThread(Thread.currentThread());
        else
            // 调用AQS的acquire方法
            acquire(1);
    }
```

我们知道AQS的acquire方法是一个模板方法，它会调用tryAcquire方法尝试获取同步状态，如果失败则会把线程加入同步队列中，通过死循环的方式获取同步状态。

NonfairSync在实现tryAcquire方法时，直接调用的Sync的nonfairTryAcquire方法，它首先获取同步状态，如果```state == 0```，表示当前没有线程获取到同步状态，然后通过CAS的方式获取同步状态，如果获取成功则设置当前线程独占同步状态并返回true；如果```state ！= 0```，则判断当前线程是否是获取到同步状态的线程，如果是，表示线程重入，增加state的值并返回true；其余情况返回false。

```
    final boolean nonfairTryAcquire(int acquires) {
        final Thread current = Thread.currentThread();
        int c = getState();
        // 没有线程获取到同步状态，当前线程尝试通过CAS获取同步状态
        if (c == 0) {
            if (compareAndSetState(0, acquires)) {
                setExclusiveOwnerThread(current);
                return true;
            }
        }
        // 如果当前线程是获取到同步状态的线程，实现重入
        else if (current == getExclusiveOwnerThread()) {
            int nextc = c + acquires;
            if (nextc < 0) // overflow
                throw new Error("Maximum lock count exceeded");
            setState(nextc);
            return true;
        }
        return false;
    }
```


#### 公平锁 FairSync

现在我们再来看看FairSync是怎么实现获取锁的，FairSync同样实现了lock方法，只不过它是直接调用了AQS的acquire方法。

```
    final void lock() {
        acquire(1);
    }
```

同样的AQS在实现acquire方法时会调用FairSync的tryAcquire方法，从FairSync的实现上来看，与NonfairSync的实现几乎一样，唯一的区别是当没有线程获取到同步状态时，FairSync多调用了一个AQS的hasQueuedPredecessors方法，这个方法的意思是同步队列中是否有其他线程排在我前面，这里分两种情况

- 同步队列是空的，表示没有其他线程排在当前线程前面，返回false
- 同步队列不为空，当前线程如果排在第一个，返回false，否则返回true


```
    protected final boolean tryAcquire(int acquires) {
        final Thread current = Thread.currentThread();
        int c = getState();
        if (c == 0) {
            // 多调用了hasQueuedPredecessors()
            if (!hasQueuedPredecessors() &&
                    compareAndSetState(0, acquires)) {
                setExclusiveOwnerThread(current);
                return true;
            }
        }
        else if (current == getExclusiveOwnerThread()) {
            int nextc = c + acquires;
            if (nextc < 0)
                throw new Error("Maximum lock count exceeded");
            setState(nextc);
            return true;
        }
        return false;
    }
```

#### 公平锁和非公平锁的区别

公平锁和非公平锁的区别在于获取锁的时候是否按照FIFO的顺序来。我们举个形象点的例子来解释它俩的区别，比如我们去银行柜台办理业务，如果发现柜台前没人，非公平锁就直接去办理，不管有没有人在排队；公平锁则是先看看有没有人排队，没人排队或者自己就是排队的第一个人再去办理。

通常来说非公平锁效率更高，在《Java并发编程的艺术》一书中做过测试(测试环境：Ubuntu Server14.04 i5-34708GB，测试场景：10个线程，每个线程获取100000次锁)，总耗时公平锁是非公平锁的94.3倍，线程上下文切换公平锁是非公平锁的133倍，所以ReentrantLock默认实现是非公平锁。

但是非公平锁容易出现同一个线程连续获取锁的情况，所以可能会出现饥饿的问题。

### 释放锁

ReentrantLock释放锁同样是委托给Sync来实现的，具体是调用了AQS的release方法，release方法最终会调用子类的tryRelease方法来释放同步状态。

```
    public void unlock() {
        sync.release(1);
    }
```

下面我们就来看下Sync具体实现tryRelease的过程，因为重入锁的原因，每释放一次锁，就是将state减去相应的值，如果释放的不是持有锁的线程则抛出异常，如果```state == 0```表示已经完全释放锁， 其他线程可以获取锁。这里可以看出释放锁的过程，公平锁和非公平锁是一样的。

```
    protected final boolean tryRelease(int releases) {
        // 重入锁 减去releases
        int c = getState() - releases;
        // 如果释放的不是持有锁的线程，抛出异常
        if (Thread.currentThread() != getExclusiveOwnerThread())
            throw new IllegalMonitorStateException();
        boolean free = false;
        // 已经完全释放，其他线程可以获取锁
        if (c == 0) {
            free = true;
            setExclusiveOwnerThread(null);
        }
        setState(c);
        return free;
    }
```



## ReentrantReadWriteLock

重入锁ReentrantLock是排他锁，排他锁在同一时刻仅有一个线程可以进行访问，但是在大多数场景下读多写少，如果一个线程在读时禁止其他线程读势必会导致性能降低，所以J.U.C就提供了读写锁。

读写锁在同一时间可以允许多个读线程同时访问，但是在写线程访问时，所有读线程和写线程都会被阻塞。读写锁维护着一对锁，一个读锁和一个写锁。通过分离读锁和写锁，使得并发性比一般的排他锁有了较大的提升。

```
public interface ReadWriteLock {
    
    Lock readLock();

    Lock writeLock();
}
```

J.U.C提供了读写锁的实现ReentrantReadWriteLock，其内部同样是基于AQS来实现的。ReentrantReadWriteLock有一个内部类Sync，Sync继承自AQS并重写了tryAcquire、tryRelease、tryAcquireShared和tryReleaseShared等方法。ReentrantReadWriteLock内部的读锁和写锁都是依靠Sync来实现的，也就是ReentrantReadWriteLock内部其实只有一把锁，只是在获取读取锁和写入锁的方式上不一样而已。

```
    // 读锁
    private final ReentrantReadWriteLock.ReadLock readerLock;
    // 写锁
    private final ReentrantReadWriteLock.WriteLock writerLock;

    final Sync sync;

    // 默认使用非公平锁
    public ReentrantReadWriteLock() {
        this(false);
    }

    // 通过指定使用公平锁或非公平锁
    public ReentrantReadWriteLock(boolean fair) {
        sync = fair ? new FairSync() : new NonfairSync();
        readerLock = new ReadLock(this);
        writerLock = new WriteLock(this);
    }

    // 非公平锁
    static final class NonfairSync extends Sync {
        // 省略其余源代码
    }

    // 公平锁
    static final class FairSync extends Sync {
        // 省略其余源代码
    }


    public ReentrantReadWriteLock.WriteLock writeLock() {
        return writerLock;
    }

    public ReentrantReadWriteLock.ReadLock readLock() {
        return readerLock;
    }

    // Sync继承自AQS
    abstract static class Sync extends AbstractQueuedSynchronizer {
        // 省略其余源代码
    }

    // 写锁
    public static class WriteLock implements Lock, java.io.Serializable {

        private final Sync sync;

        // 内部使用Sync来实现
        protected WriteLock(ReentrantReadWriteLock lock) {
            sync = lock.sync;
        }

        // 省略其余源代码
    }

    // 读锁
    public static class ReadLock implements Lock, java.io.Serializable {

        private final Sync sync;

        // 内部使用Sync来实现
        protected WriteLock(ReentrantReadWriteLock lock) {
            sync = lock.sync;
        }

        // 省略其余源代码
    }
```

在ReentrantLock中使用一个int类型的state来表示同步状态，该值表示锁被一个线程重复获取的次数。但是读写锁ReentrantReadWriteLock内部维护着两个一对锁，需要用一个变量维护多种状态。所以读写锁采用“按位切割使用”的方式来维护这个变量，将其切分为两部分，高16为表示读，低16为表示写。分割之后，读写锁是如何迅速确定读锁和写锁的状态呢？通过为运算。假如当前同步状态为S，那么写状态等于 S & 0x0000FFFF（将高16位全部抹去），读状态等于S >>> 16(无符号补0右移16位)。代码如下：

```
    static final int SHARED_SHIFT   = 16;
    static final int SHARED_UNIT    = (1 << SHARED_SHIFT);
    static final int MAX_COUNT      = (1 << SHARED_SHIFT) - 1;
    static final int EXCLUSIVE_MASK = (1 << SHARED_SHIFT) - 1;

    static int sharedCount(int c)    { return c >>> SHARED_SHIFT; }

    static int exclusiveCount(int c) { return c & EXCLUSIVE_MASK; }
```

ReentrantReadWriteLock内部具体的实现我们这里就不做展开了，如果读者感兴趣可以去阅读源码。



### 用法示例

ReentrantReadWriteLock内部存在一把读锁和一把写锁，他们的用法和Lock的用法一样。

```
    private static final ReadWriteLock lock = new ReentrantReadWriteLock();

    private static final Lock readLock = lock.readLock();

    private static final Lock writeLock = lock.writeLock();

    private static Map<Integer, Integer> map = new HashMap<>();

    public void write(int threadNum) {
        writeLock.lock();
        try {
            log.info("write threadNum start : {}", threadNum);
            map.put(threadNum, threadNum);
        } finally {
            writeLock.unlock();
            log.info("write threadNum end : {}", threadNum);
        }
    }

    public void read(int threadNum) {
        readLock.lock();
        try {
            log.info("read threadNum start : {}", threadNum);
            map.get(threadNum);
        } finally {
            readLock.unlock();
            log.info("read threadNum end : {}", threadNum);
        }
    }
```

### 优缺点

ReentrantReadWriteLock由于读线程不相互阻塞，所以一般情况下相比较于ReentrantLock有更高的效率，但是如果读线程特别多而写线程特别少的情况下，写线程可能一直获取不到锁，从而发生饥饿的现象。



## StampedLock

StampedLock是Java1.8提供一个新锁，StampedLock控制锁有三种模式（写，读，乐观读），一个StampedLock状态是由版本和模式两个部分组成，锁获取方法返回一个数字作为票据stamp，它用相应的锁状态表示并控制访问，数字0表示没有写锁被授权访问。在读锁上分为悲观锁和乐观锁。

### 写锁

StampedLock的写锁是个排它锁或者叫独占锁，同时只有一个线程可以获取该锁，当一个线程获取该锁后，其它请求的线程必须等待，当目前没有线程持有读锁或者写锁的时候才可以获取到该锁，请求该锁成功后会返回一个stamp票据变量用来表示该锁的版本，当释放该锁时候需要unlockWrite并传递参数stamp。

写锁的用法如下所示

```
    private static final StampedLock lock = new StampedLock();
    
    public void write() {
        long stamp = lock.writeLock();
        try {
            doWrite();
        } finally {
            lock.unlock(stamp);
        }
    }
```

### 读锁

StampedLock的读锁是共享锁，但是它是悲观读，也就是假设在读数据的过程中会有其他线程写数据，所以读锁会阻塞其他写线程，但是不会阻塞其他的读线程。这是在读少写多的情况下的一种考虑,请求该锁成功后会返回一个stamp票据变量用来表示该锁的版本，当释放该锁时候需要unlockRead并传递参数stamp。

读锁的用法如下所示

```
    public void read() {
        long stamp = lock.readLock();
        try {
            doRead();
        } finally {
            lock.unlock(stamp);
        }
    }
```

### 乐观读

所谓的乐观读，也就是若读的操作很多，写的操作很少的情况下，你可以乐观地认为，写入与读取同时发生几率很少，因此不悲观地使用完全的读取锁定，程序可以查看读取资料之后，是否遭到写入执行的变更，再采取后续的措施（重新读取变更信息，或者抛出异常） ，这一个小小改进，可大幅度提高程序的吞吐量。

乐观读的用法如下所示

```
    public void optimisticRead() {
        long stamp = lock.tryOptimisticRead();
        doRead();
        // 读的过程中数据被改变，则使用读锁
        if(!lock.validate(stamp)){
            long stamp = lock.readLock();
            try {
                doRead();
            } finally {
                lock.unlock(stamp);
            }
        }
    }
```

## 总结

现在我们学习了四种锁的实现：synchronized、ReentrantLock、ReentrantReadWriteLock和StampedLock，那么我们要实现锁时应该怎么选择呢？

- 通常情况下synchronized是一个很好的选择，因为synchronized对于开发人员来说很熟悉，且JVM会自动释放锁。仅当synchronized不能满足需求时（比如竞争激烈时效率低下或者必须使用显示锁的一些特性时），再考虑使用其他显示锁。
- 使用显示锁时，ReentrantLock通常是一个很好的选择，因为它使用简单，容易理解，不易出错。
- ReentrantReadWriteLock和StampedLock性能更优，相对应的它们的使用也越来越复杂，如果不是必须，尽量不要使用。
- 在合适的场景使用合适的锁，而不是尽量使用更高级的锁。
- 锁尽量不要混用，否则代码难以理解，也更易出错。
