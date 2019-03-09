---
title: HashMap源码阅读
categories:
  - [源码]
  - [Java]
tags:
  - 源码
  - Java
date: 2018-11-13 21:42:38
---

Map是一种键值对的数据结构，而HashMap是我们最常用的一种实现，<!-- more -->它的存取速度很快，元素无序，线程不安全。本文会首先介绍下HashMap的基本原理，然后再阅读源码。


> 本篇文章源码基于 JDK 1.8


## HashMap的实现原理

HashMap的底层结构是数组 + 链表，数组中的每一个位置我们称之为桶位，对于每一个Key-Value(KV)，首先会计算K的hash值，然后通过对数组长度取余（实际上是通过 ```&``` 操作）获取元素的桶位，如果多个KV分配到了相同的桶位，那么它们之间就会组成链表，当然在 JDK 1.8 中，当链表的长度大于8时，链表会转化为红黑树来提高查询效率。


## 源码阅读

简单了解了HashMap的实现原理后，我们就来一起学习下源码吧。

### 常量

```
    /**
     * 默认初始容量 16
     */
    static final int DEFAULT_INITIAL_CAPACITY = 1 << 4; 

    /**
     * 最大容量
     */
    static final int MAXIMUM_CAPACITY = 1 << 30;

    /**
     * 默认负载因子
     */
    static final float DEFAULT_LOAD_FACTOR = 0.75f;

    /**
     * 当桶位上的链表长度大于 TREEIFY_THRESHOLD 时，链表会转化为红黑树
     */
    static final int TREEIFY_THRESHOLD = 8;

    /**
     * 当红黑树元素个数小于 UNTREEIFY_THRESHOLD 时，红黑树会转化为链表
     */
    static final int UNTREEIFY_THRESHOLD = 6;

    /**
     * 链表转化为红黑树的另一个条件，就是整个HashMap的容量至少要大于 MIN_TREEIFY_CAPACITY
     */
    static final int MIN_TREEIFY_CAPACITY = 64;

```


### 成员变量

```
    /**
     * 底层存储元素的列表，列表的长度称之为 capacity
     */
    transient Node<K,V>[] table;

    /**
     * 用于缓存内部元素
     */
    transient Set<Map.Entry<K,V>> entrySet;

    /**
     * 内部元素的 size
     */
    transient int size;

    /**
     * 操作次数
     */
    transient int modCount;

    /**
     * 扩容的阈值，当 HashMap 的 size > threshold 时会进行扩容
     * 一般情况下 threshold = capacity * loadFactor
     * 对于刚初始化的 HashMap 来说，它可能是 0 或者 capacity
     */
    int threshold;

    /**
     * 负载因子
     */
    final float loadFactor;

```

### 重要内部类

#### Node

在 HashMap 内部，每一对 KV 都会组装成一个 Node 保存。

```
    static class Node<K,V> implements Map.Entry<K,V> {
        // key的hash值
        final int hash;
        final K key;
        V value;
        // 相同桶位的Node会组成链表，指向下一个Node
        Node<K,V> next;

        Node(int hash, K key, V value, Node<K,V> next) {
            this.hash = hash;
            this.key = key;
            this.value = value;
            this.next = next;
        }

        public final K getKey()        { return key; }
        public final V getValue()      { return value; }
        public final String toString() { return key + "=" + value; }

        public final int hashCode() {
            return Objects.hashCode(key) ^ Objects.hashCode(value);
        }

        public final V setValue(V newValue) {
            V oldValue = value;
            value = newValue;
            return oldValue;
        }

        public final boolean equals(Object o) {
            if (o == this)
                return true;
            if (o instanceof Map.Entry) {
                Map.Entry<?,?> e = (Map.Entry<?,?>)o;
                if (Objects.equals(key, e.getKey()) &&
                    Objects.equals(value, e.getValue()))
                    return true;
            }
            return false;
        }
    }
```

### 构造函数

```
    /**
     * 指定初始容量和负载因子
     */
    public HashMap(int initialCapacity, float loadFactor) {
        if (initialCapacity < 0)
            throw new IllegalArgumentException("Illegal initial capacity: " +
                                               initialCapacity);
        if (initialCapacity > MAXIMUM_CAPACITY)
            initialCapacity = MAXIMUM_CAPACITY;
        if (loadFactor <= 0 || Float.isNaN(loadFactor))
            throw new IllegalArgumentException("Illegal load factor: " +
                                               loadFactor);
        this.loadFactor = loadFactor;
        // 计算初始的扩容阈值，实际上是初始容量
        this.threshold = tableSizeFor(initialCapacity);
    }

    /**
     * 指定初始容量，使用默认负载因子
     */
    public HashMap(int initialCapacity) {
        this(initialCapacity, DEFAULT_LOAD_FACTOR);
    }

    /**
     * 使用默认容量和默认负载因子
     */
    public HashMap() {
        this.loadFactor = DEFAULT_LOAD_FACTOR;
    }
```

在第一个构造函数中，最后一句很重要，它是计算实际的初始容量，因为 HashMap 要求容量必须为 2 的 n 次方，这样的好处是可以把取余操作转化为速度更快的按位与。比如容量为16，某个 key 的 hash 值是40，计算桶位的话可以通过取余操作```40 % 16 = 8```；当然我们也可以使用按位与```(16 - 1) & 40 = 8```，这种转化的前提是容量必须是 2 的 n 次方。

刚才我们说了HashMap 要求容量必须为 2 的 n 次方，但是用户传入的初始容量不可能都是符合要求的，所以要对用户传入的容量做进一步处理，比如用户传入了5，则 HashMap 的初始容量实际上是8；用户传入了13，则 HashMap 的初始容量实际上是16。而完成这种转换的就是函数```tableSizeFor(int cap)```。

```
    static final int tableSizeFor(int cap) {
        int n = cap - 1;
        n |= n >>> 1;
        n |= n >>> 2;
        n |= n >>> 4;
        n |= n >>> 8;
        n |= n >>> 16;
        return (n < 0) ? 1 : (n >= MAXIMUM_CAPACITY) ? MAXIMUM_CAPACITY : n + 1;
    }
```

### 重要方法

#### put

```
    /**
     * 添加元素
     */
    public V put(K key, V value) {
        return putVal(hash(key), key, value, false, true);
    }

    /**
     * Implements Map.put and related methods
     *
     * @param hash hash for key
     * @param key the key
     * @param value the value to put
     * @param onlyIfAbsent if true, don't change existing value
     * @param evict if false, the table is in creation mode.
     * @return previous value, or null if none
     */
    final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
                   boolean evict) {
        Node<K,V>[] tab; Node<K,V> p; int n, i;
        // 如果 table 还未初始化，则通过 resize() 初始化
        if ((tab = table) == null || (n = tab.length) == 0)
            n = (tab = resize()).length;
        // 计算元素所在桶位，如果桶位为null，说明该桶位还未有任何元素，则直接新建一个 Node 放在该桶位上    
        if ((p = tab[i = (n - 1) & hash]) == null)
            tab[i] = newNode(hash, key, value, null);
        else {
            // 桶位上有元素
            
            // e 表示保存插入元素的Node
            Node<K,V> e; K k;
            // 如果桶位上元素和要插入元素的 key 是相同的，则直接保存在桶位的 Node 上
            if (p.hash == hash &&
                ((k = p.key) == key || (key != null && key.equals(k))))
                e = p;
            // 如果桶位上 Node 是 TreeNode，说明该桶位上是一颗红黑树，则调用 TreeNode 方法插入元素   
            else if (p instanceof TreeNode)
                e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
            else {
                // 桶位上是一个链表，则循环判断链表上的每一个元素
                for (int binCount = 0; ; ++binCount) {
                    // 循环到最后一个元素都没有 key 相等，则新建一个 Node 追加在链表最后
                    if ((e = p.next) == null) {
                        p.next = newNode(hash, key, value, null);
                        // 链表长度大于等于 TREEIFY_THRESHOLD，链表转化为红黑树
                        if (binCount >= TREEIFY_THRESHOLD - 1)
                            treeifyBin(tab, hash);
                        break;
                    }
                    // 如果元素和要插入元素的 key 是相同的，则保存在该 Node 上
                    if (e.hash == hash &&
                        ((k = e.key) == key || (key != null && key.equals(k))))
                        break;
                    p = e;
                }
            }
            // e 不为空，则替换 value 值
            if (e != null) {
                V oldValue = e.value;
                if (!onlyIfAbsent || oldValue == null)
                    e.value = value;
                afterNodeAccess(e);
                return oldValue;
            }
        }
        ++modCount;
        if (++size > threshold)
            // 进行扩容
            resize();
        afterNodeInsertion(evict);
        return null;
    }
``` 


#### get

```
    public V get(Object key) {
        Node<K,V> e;
        return (e = getNode(hash(key), key)) == null ? null : e.value;
    }

    /**
     * Implements Map.get and related methods
     *
     * @param hash hash for key
     * @param key the key
     * @return the node, or null if none
     */
    final Node<K,V> getNode(int hash, Object key) {
        Node<K,V>[] tab; Node<K,V> first, e; int n; K k;
        // table 不为空，并且 key 对应的桶位有值
        if ((tab = table) != null && (n = tab.length) > 0 &&
            (first = tab[(n - 1) & hash]) != null) {
            // 检查桶位上的 Node 是否满足
            if (first.hash == hash &&
                ((k = first.key) == key || (key != null && key.equals(k))))
                return first;
            if ((e = first.next) != null) {
                // 如果是 TreeNode 则使用 TreeNode.getTreeNode
                if (first instanceof TreeNode)
                    return ((TreeNode<K,V>)first).getTreeNode(hash, key);
                // 否则轮询链表上的每一个 Node    
                do {
                    if (e.hash == hash &&
                        ((k = e.key) == key || (key != null && key.equals(k))))
                        return e;
                } while ((e = e.next) != null);
            }
        }
        return null;
    }
```


#### 扩容

```
   /**
     * 初始化 table 或者将 table 的容量扩展为原来的2倍
     *
     * @return the table
     */
    final Node<K,V>[] resize() {
        Node<K,V>[] oldTab = table;
        int oldCap = (oldTab == null) ? 0 : oldTab.length;
        int oldThr = threshold;
        int newCap, newThr = 0;
        // table 不为空，容量扩展为原来的2倍
        if (oldCap > 0) {
            if (oldCap >= MAXIMUM_CAPACITY) {
                threshold = Integer.MAX_VALUE;
                return oldTab;
            }
            else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY &&
                     oldCap >= DEFAULT_INITIAL_CAPACITY)
                newThr = oldThr << 1;
        }
        // 通过 HashMap(int initialCapacity, float loadFactor) 或者 HashMap(int initialCapacity) 构造的 HashMap 会走这一步
        else if (oldThr > 0)
            newCap = oldThr;
        // 通过 HashMap() 构造的 HashMap 会走这一步    
        else {
            newCap = DEFAULT_INITIAL_CAPACITY;
            newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
        }
        // 如果新的阈值等于0，则计算
        if (newThr == 0) {
            float ft = (float)newCap * loadFactor;
            newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ?
                      (int)ft : Integer.MAX_VALUE);
        }
        threshold = newThr;
        // 新的数组
        Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
        table = newTab;
        // 将旧数组中的数据重新分布到新的数组中
        if (oldTab != null) {
            for (int j = 0; j < oldCap; ++j) {
                Node<K,V> e;
                if ((e = oldTab[j]) != null) {
                    oldTab[j] = null;
                    if (e.next == null)
                        newTab[e.hash & (newCap - 1)] = e;
                    else if (e instanceof TreeNode)
                        ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
                    else { // preserve order
                        Node<K,V> loHead = null, loTail = null;
                        Node<K,V> hiHead = null, hiTail = null;
                        Node<K,V> next;
                        do {
                            next = e.next;
                            if ((e.hash & oldCap) == 0) {
                                if (loTail == null)
                                    loHead = e;
                                else
                                    loTail.next = e;
                                loTail = e;
                            }
                            else {
                                if (hiTail == null)
                                    hiHead = e;
                                else
                                    hiTail.next = e;
                                hiTail = e;
                            }
                        } while ((e = next) != null);
                        if (loTail != null) {
                            loTail.next = null;
                            newTab[j] = loHead;
                        }
                        if (hiTail != null) {
                            hiTail.next = null;
                            newTab[j + oldCap] = hiHead;
                        }
                    }
                }
            }
        }
        return newTab;
    }
```
