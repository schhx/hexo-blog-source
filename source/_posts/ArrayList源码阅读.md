---
title: ArrayList源码阅读
categories:
  - [源码]
  - [Java]
tags:
  - 源码
  - Java
date: 2018-11-07 21:19:40
---

ArrayList是一个非常常用的集合类，线程不安全，它实现了List接口，底层使用数组来存储数据，读取速度比较快，增删速度比较慢。<!-- more --> 下面我们就来学习下ArrayList的源码。

> 本篇文章源码基于 JDK 1.8

### 常量

```
    /**
     * 默认初始化容量.
     */
    private static final int DEFAULT_CAPACITY = 10;

    /**
     * 默认空数组，初始化数组时，如果指定初始容量为0，则 elementData = EMPTY_ELEMENTDATA.
     */
    private static final Object[] EMPTY_ELEMENTDATA = {};

    /**
     * 默认空数组，初始化数组时，如果不指定初始容量，则 elementData = DEFAULTCAPACITY_EMPTY_ELEMENTDATA.
     */
    private static final Object[] DEFAULTCAPACITY_EMPTY_ELEMENTDATA = {};
    
    /**
     * 列表允许的最大容量
     */
    private static final int MAX_ARRAY_SIZE = Integer.MAX_VALUE - 8;
```

### 成员变量

```   
    /**
     * 底层存储结构.
     */
    transient Object[] elementData;
    
    /**
     * 长度.
     */
    private int size;
```

从上面的代码中我们可以看到有两个默认的空数组 EMPTY_ELEMENTDATA 和 DEFAULTCAPACITY_EMPTY_ELEMENTDATA，这两个空数组对应着两种情况：

- 初始化列表时，如果指定初始容量为0，则使用 EMPTY_ELEMENTDATA。
- 初始化列表时，如果不指定初始容量，那么应该使用默认容量 DEFAULT_CAPACITY，但是为了节省空间，实际上刚初始化出来的数组其实是一个空数组 DEFAULTCAPACITY_EMPTY_ELEMENTDATA，当第一个元素添加的时候，才会把数组扩容到 DEFAULT_CAPACITY。

下文用 **默认容量列表** 指代使用无参构造函数构造、还未添加任何元素的列表。

### 构造函数

```
    /**
     * 使用指定容量构造
     */
    public ArrayList(int initialCapacity) {
        if (initialCapacity > 0) {
            // 按指定容量构造
            this.elementData = new Object[initialCapacity];
        } else if (initialCapacity == 0) {
            // 指定容量为0，使用 EMPTY_ELEMENTDATA
            this.elementData = EMPTY_ELEMENTDATA;
        } else {
            throw new IllegalArgumentException("Illegal Capacity: "+
                                               initialCapacity);
        }
    }

    /**
     * 不指定容量，使用默认容量构造.
     */
    public ArrayList() {
        // 不指定容量，使用 DEFAULTCAPACITY_EMPTY_ELEMENTDATA
        this.elementData = DEFAULTCAPACITY_EMPTY_ELEMENTDATA;
    }

    /**
     * 通过集合类构造
     */
    public ArrayList(Collection<? extends E> c) {
        elementData = c.toArray();
        if ((size = elementData.length) != 0) {
            // c.toArray might (incorrectly) not return Object[] (see 6260652)
            if (elementData.getClass() != Object[].class)
                elementData = Arrays.copyOf(elementData, size, Object[].class);
        } else {
            this.elementData = EMPTY_ELEMENTDATA;
        }
    }

```

这里我们重点看一下第三个构造函数，这个里面有一个地方要注意下，c.toArray 可能返回的并不是 Object[] 类型，如果不做处理的话，再添加元素就会报错。这边可以用一个简单的例子说明，比如如下代码：

```
    public static void main(String[] args) {
        String[] strings = new String[]{"a", "b", "c"};
        Object[] objects = strings;
        System.out.println(objects.getClass());
        objects[0] = new Object();
    }
``` 

运行结果：

```
class [Ljava.lang.String;
Exception in thread "main" java.lang.ArrayStoreException: java.lang.Object
	at com.wosai.app.backend.api.service.ss.main(ss.java:13)
```

我们可以看出，虽然看起来 objects 是 Object[] 类型，但实际上是 String[] 类型，当我们给 objects[0] 赋值 new Object() 时就会出现向下转型异常。也就是说假如我们有 1 个 Object[] 数组，并不代表着我们可以将 Object 对象存进去，这取决于数组中元素实际的类型。


### 重要方法


#### 追加元素

```
    public boolean add(E e) {
        // 保证容量充足，如有必要会进行扩容
        ensureCapacityInternal(size + 1);
        elementData[size++] = e;
        return true;
    }
```

这个方法组重要的一句就是```ensureCapacityInternal(size + 1)```，关于这个方法下面会详细讲解。


#### 替换元素

```
    /**
     * 在指定位置替换元素
     */
    public E set(int index, E element) {
        rangeCheck(index);

        E oldValue = elementData(index);
        elementData[index] = element;
        return oldValue;
    }
    
    /**
     * 检查index是否在合理范围内
     */
    private void rangeCheck(int index) {
        if (index >= size)
            throw new IndexOutOfBoundsException(outOfBoundsMsg(index));
    }    
```


#### 扩容

```
    /**
     * 保证容量能够支持 minCapacity
     *
     * @param   minCapacity   期待最小容量
     */
    public void ensureCapacity(int minCapacity) {
        // 最小需要扩展的容量
        // 默认容量列表，则最小应扩容到 DEFAULT_CAPACITY
        // 其他情况，最小需扩展的容量为 0
        int minExpand = (elementData != DEFAULTCAPACITY_EMPTY_ELEMENTDATA) ? 0 : DEFAULT_CAPACITY;

        if (minCapacity > minExpand) {
            ensureExplicitCapacity(minCapacity);
        }
    }

    private void ensureCapacityInternal(int minCapacity) {
        // 如果是默认容量列表，则扩展到 Math.max(DEFAULT_CAPACITY, minCapacity)
        if (elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA) {
            minCapacity = Math.max(DEFAULT_CAPACITY, minCapacity);
        }

        ensureExplicitCapacity(minCapacity);
    }

    private void ensureExplicitCapacity(int minCapacity) {
        modCount++;

        // minCapacity 大于当前容量，则需要扩容
        if (minCapacity - elementData.length > 0)
            grow(minCapacity);
    }
    
    /**
     * 具体扩容操作
     */
    private void grow(int minCapacity) {
        int oldCapacity = elementData.length;
        // 新容量为老容量的1.5倍
        int newCapacity = oldCapacity + (oldCapacity >> 1);
        // 如果扩容了1.5倍还不够用，则直接将新容量扩容到 minCapacity
        if (newCapacity - minCapacity < 0)
            newCapacity = minCapacity;
        // 如果新容量大于列表允许的最大容量，则调用 hugeCapacity 方法确定新容量
        if (newCapacity - MAX_ARRAY_SIZE > 0)
            newCapacity = hugeCapacity(minCapacity);
        // 拷贝数组
        elementData = Arrays.copyOf(elementData, newCapacity);
    }
    
    private static int hugeCapacity(int minCapacity) {
        if (minCapacity < 0)
            throw new OutOfMemoryError();
        return (minCapacity > MAX_ARRAY_SIZE) ?
            Integer.MAX_VALUE :
            MAX_ARRAY_SIZE;
    }
```

扩容方法比较重要，也比较耗时，如果我们在代码中知道要添加大量元素，可以构造列表时指定较大的容量，或者在添加元素之前手动调用 ensureCapacity 方法扩容，这会提高代码的运行效率。

#### 缩容

```
    public void trimToSize() {
        modCount++;
        if (size < elementData.length) {
            elementData = (size == 0)
              ? EMPTY_ELEMENTDATA
              : Arrays.copyOf(elementData, size);
        }
    }
```
