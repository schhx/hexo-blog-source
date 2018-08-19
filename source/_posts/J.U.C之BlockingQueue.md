---
title: J.U.C之BlockingQueue
categories:
  - Java并发编程
tags:
  - Java并发编程
date: 2018-08-07 22:15:08
---

阻塞队列(BlockingQueue)，顾名思义，它首先是一个队列，然后这个队列可能会对插入和移除操作阻塞，具体可以分为以下两种情况：<!-- more -->

- 当队列已满的时候，插入操作会被阻塞
- 当队列为空的时候，移除操作会被阻塞


## 四套操作方法

阻塞队列常用于生产者消费者场景，生产者向队列里插入元素，消费者从队列里移除元素。阻塞队列提供了四套插入和移除的方法，这四套方法在阻塞队列不可用时，表现出来的行为并不一致，具体如下：


方法 | 抛出异常 | 返回特殊值 | 阻塞 | 超时返回特殊值
---|---|---|---|---
插入方法 | add(e) | offer(e) | put(e) | offer(e, timeout, unit)
移除方法 | remove() | poll() | take() | poll(timeout, unit)
检查方法 | element() | peek() | 无 | 无



这里说一下检查方法，检查方法和移除方法类似，都是从队列中取出一个元素，但是检查方法并不会把取出的元素从队列中移除。

下面我们具体介绍一下这些方法：

- boolean add(E e) : 插入成功返回true，插入失败抛出 IllegalStateException
- E remove() : 移除并返回队列头部的元素，队列不为空时返回移除的元素，队列为空时抛出 NoSuchElementException
- E element() : 返回队列头部的元素，但并不移除，成功返回头部元素，失败抛出 NoSuchElementException
- boolean offer(E e) : 插入成功返回true，插入失败返回false
- E poll() : 移除并返回队列头部的元素，队列不为空时返回移除的元素，队列为空时返回null
- E peek() : 返回队列头部的元素，但并不移除，成功返回头部元素，失败返回null
- void put(E e) : 队列为空时插入元素，队列已满时线程会被阻塞，直到队列有空余位置或线程被中断
- E take() : 移除并返回队列头部的元素，队列不为空时返回移除的元素，队列为空时线程会被阻塞，直到队列有空余位置或线程被中断
- boolean offer(E e, long timeout, TimeUnit unit) : offer(e)的超时版本
- E poll(long timeout, TimeUnit unit) : poll()的超时版本


## 具体实现类

### ArrayBlockingQueue

ArrayBlockingQueue是一个由数组实现的有界阻塞队列，该队列采用FIFO的原则对元素进行出队和入队操作。ArrayBlockingQueue为有界且固定，其大小在构造时由构造函数来决定，确认之后就不能再改变了。ArrayBlockingQueue支持对等待的生产者线程和使用者线程进行排序的可选公平策略，但是在默认情况下不保证线程公平的访问，在构造时可以选择公平策略（fair = true）。公平性通常会降低吞吐量，但是减少了可变性和避免了“不平衡性”。

### LinkedBlockingQueue

LinkedBlockingQueue是一个由链表实现的有界阻塞队列，该队列同样采用FIFO的原则对元素进行出队和入队操作。LinkedBlockingQueue可以在初始化时指定容量，如果不指定，默认容量是Integer.MAX_VALUE。

### LinkedBlockingDeque

LinkedBlockingDeque则是一个由链表组成的双向阻塞队列，双向队列就意味着可以从对头、对尾两端插入和移除元素，同样意味着LinkedBlockingDeque支持FIFO、FILO两种操作方式。LinkedBlockingDeque是可选容量的，在初始化时可以设置容量防止其过度膨胀，如果不设置，默认容量大小为Integer.MAX_VALUE。

### PriorityBlockingQueue

PriorityBlockingQueue是一个支持优先级的无界阻塞队列。默认情况下元素采用自然顺序升序排序，当然我们也可以通过构造函数来指定Comparator来对元素进行排序。需要注意的是PriorityBlockingQueue不能保证同优先级元素的顺序。

### DelayQueue

DelayQueue是一个支持延时获取元素的无界阻塞队列。里面的元素全部都是“可延期”的元素，列头的元素是最先“到期”的元素，如果队列里面没有元素到期，是不能从列头获取元素的，哪怕有元素也不行。也就是说只有在延迟期到时才能够从队列中取元素。

### SynchronousQueue

作为BlockingQueue中的一员，SynchronousQueue与其他BlockingQueue有着不同特性：

- SynchronousQueue没有容量。与其他BlockingQueue不同，SynchronousQueue是一个不存储元素的BlockingQueue。每一个put操作必须要等待一个take操作，否则不能继续添加元素，反之亦然。
- 因为没有容量，所以对应 peek, contains, clear, isEmpty ... 等方法其实是无效的。例如clear是不执行任何操作的，contains始终返回false,peek始终返回null。
- SynchronousQueue分为公平和非公平，默认情况下采用非公平性访问策略，当然也可以通过构造函数来设置为公平性访问策略（为true即可）。

### LinkedTransferQueue

LinkedTransferQueue实现的是TransferQueue接口，而TransferQueue接口又继承自BlockingQueue，LinkedTransferQueue是基于链表的FIFO无界阻塞队列。TransferQueue相对于BlockingQueue增加3个比较重要的方法：

- void transfer(E e): 如果当前有消费者等待接收元素(消费者使用take()或poll(timeout, unit)方法)，生产者可以通过transfer(E e)方法直接把元素给消费者。如果没有消费者等待，transfer(E e)方法会把元素加到队列尾部，并等到改元素被消费者消费了再返回。
- boolean tryTransfer(E e): 用来尝试元素能不能直接传给消费者，如果有消费者等待，则直接把元素传给消费者，然后返回true，否则直接返回false
- boolean tryTransfer(E e, long timeout, TimeUnit unit): tryTransfer(E e)的超时版本








