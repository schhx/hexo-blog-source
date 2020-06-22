---
title: Spring学习笔记--什么是IoC
categories:
  - Spring学习笔记
tags:
  - Spring学习笔记
date: 2020-04-21 15:26:55
---

IoC（Inversion of Control）的意思是控制反转，那反转的是什么东西呢？<!-- more -->反转的是对象创建的过程，由客户端创建依赖的对象反转为外部创建对象，客户端直接使用创建好的对象。通常和 IoC 一起出现的另一个概念是 DI (Dependency Injection，依赖注入)，很多人认为 Ioc 和 DI 是一回事，如果不严格区分的话，可以把 IoC 和 DI 等同起来，但是如果严格区分的话，DI 其实只是 IoC 的一种实现方式，IoC 的实现方式有很多，除了 DI 外，还有另一种常见的实现方式 DL (Dependency Lookup，依赖查找)。

## 依赖查找

依赖查找是客户端主动通过 IoC 容器 API 来获取依赖对象，这种方式是比较传统的实现方式，其实上篇文章我们就是使用的这种方式，我们再来回顾一下。

```
/**
 * IoC 容器
 */
public class BeanFactory {

    public static <T> T getBean(Class<T> requiredType) {
        // 具体实现省略
        return null;
    }
}
```

然后客户端通过容器API主动获取依赖的对象：

```
    public Client() {
        this.strategy = BeanFactory.getBean(StockStrategy.class);
    }
```


## 依赖注入

依赖查找是客户端主动通过 IoC 容器的 API 获取依赖的对象，而依赖注入是客户端通过某种方式告诉容器我需要什么，然后由容器注入到客户端中。依赖注入通常有三种实现方式：构造方法注入、setter方法注入、接口注入。

### 构造方法注入

构造方法注入，就是被注入对象可以通过在其构造方法中声明依赖对象的参数列表， 让IoC容器知道它需要哪些依赖对象。

```
    public Client(StockStrategy strategy) {
        this.strategy = strategy;
    }
```

### setter方法注入

setter方法注入，就是让IoC容器在创建对象完成后通过setter方法把依赖的对象注入进来。

```
    public void setStrategy(StockStrategy strategy) {
        this.strategy = strategy;
    }
```

### 接口注入

接口注入，就是通过实现某个接口来注入依赖的对象。这种方式相比较前两种方式来讲比较繁琐，首先要提供一个用于注入得接口，

```
/**
 * 用来注入 StockStrategy
 */
public interface StockStrategyProvider {

    void injectStockStrategy(StockStrategy strategy);
}
```

然后客户端通过实现接口来获得依赖对象

```
public class Client implements StockStrategyProvider {
    private StockStrategy strategy;

    public void analyze() {
        getStocks().stream().forEach(stock -> doAnalyze(stock));
    }

    private void doAnalyze(String stock) {
        if (strategy.buy(stock)) {
            System.out.println("buy " + stock);
        } else if (strategy.sell(stock)) {
            System.out.println("sell " + stock);
        }
    }

    public List<String> getStocks() {
        return Arrays.asList();
    }

    @Override
    public void injectStockStrategy(StockStrategy strategy) {
        this.strategy = strategy;
    }
}
```

### 三种注入方式比较

- 接口注入：这种方式使用比较繁琐，而且强制客户端实现不必要接口，带有侵入性，所以不推荐使用，基本处于退役状态。
- 构造方法注入：这种方式的好处是对象创建完成就处于就绪状态，可以马上使用；但是当依赖对象较多时，使用上也比较麻烦，而且不能解决循环依赖的问题。
- setter方法注入：这种方式的好处是侵入性低、容易理解和使用，并且能够解决循环依赖的问题；缺点是创建完成后不能马上使用，必须等到依赖对象注入完成后才能使用。


## 依赖查找 or 依赖注入

依赖查找和依赖注入都是 IoC 的实现方式，那他们各有什么优缺点呢？

- 依赖查找是利用 IoC 容器的 API 主动获取依赖对象，所以会强依赖容器 API，对业务代码有侵入性，但是这种方式的好处是可读性良好。
- 依赖注入是由 IoC 容器把依赖注入到客户端，代码侵入性低、业务方代码比较简洁，但是代码可读性一般。

目前主流的 IoC 容器都会同时支持依赖查找和依赖注入，我们大部分情况下都是使用依赖注入的方式，以至于很多人都不太了解依赖查找，以为 IoC 和 DI 是一回事。




---

![](http://ww3.sinaimg.cn/large/0082lgKxgy1gdhu6adriej31hb0hqace.jpg)