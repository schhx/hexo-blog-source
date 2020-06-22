---
title: Spring学习笔记--为什么需要IoC
categories:
  - Spring学习笔记
tags:
  - Spring学习笔记
date: 2020-04-21 15:19:48
---

IoC（Inversion of Control，控制反转）是 Spring 提供的核心功能，Spring 为什么会提供这样的功能，开发者又为什么需要这样的功能呢？<!-- more --> 本篇文章会从一个简单的例子出发说明 IoC 是我们必然的选择。


## 一个例子

假设现在我们要完成一个分析股票的小工具，它会告诉我们什么时候买入、什么时候卖出。这个小工具包含以下几个部分:

**股票买卖策略接口**

```
/**
 * 股票买卖策略
 */
public interface StockStrategy {

    /**
     * 是否买入
     *
     * @param stock 股票代码
     * @return true 买入
     */
    boolean buy(String stock);

    /**
     * 是否卖出
     *
     * @param stock 股票代码
     * @return true 卖出
     */
    boolean sell(String stock);
}
```

**一个策略实现类**

```
/**
 * 具体策略
 *
 */
public class StockStrategyImpl implements StockStrategy {

    @Override
    public boolean buy(String stock) {
        return false;
    }

    @Override
    public boolean sell(String stock) {
        return false;
    }
}
```

**一个客户端**

```
/**
 * 客户端
 */
public class Client {
    private StockStrategy strategy;

    public Client() {
        this.strategy = new StockStrategyImpl();
    }

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
}
```

## 存在的问题

这样我们就完成了一个简易的分析股票的功能，这样的代码能够完成功能，但是由于```Client```和```StockStrategyImpl```耦合在一起，就会导致代码可测试性差、可扩展性差、难以维护。

### 可测试性差

如果我们要对```Client```做单元测试，实际上是比较困难，因为```Client```和```StockStrategyImpl```耦合在一起，我们必须要了解```StockStrategyImpl```的内部实现才能合理地构造测试用例，如果```StockStrategyImpl```还依赖了其他的组件我们可能还要了解其依赖的组件，这样会导致我们很难做单元测试。

### 可扩展性差

如果我们实现了一种新策略 ```StockStrategyImpl2```：

```
/**
 * 另一个策略
 */
public class StockStrategyImpl2 implements StockStrategy {

    @Override
    public boolean buy(String stock) {
        return false;
    }

    @Override
    public boolean sell(String stock) {
        return false;
    }
}
```

想让客户端使用新策略时，我们不得不修改客户端代码，如果只改动一个地方还好，如果有几十个客户端，那么就需要改造几十个地方，这样就使得代码的扩展性变差。

```
    public Client() {
        this.strategy = new StockStrategyImpl2();
    }
```


## 解决问题

出现上述问题的原因是客户端与具体实现类耦合，为什么会产生耦合？是因为客户端自己实例化了依赖的对象，所以解决耦合的关键就是客户端不要去实例化依赖的对象，而是应该从某个地方获取已经实例化好的对象，这就是 IoC 的思想。具体怎么做呢？其实很容易想到的方法是使用工厂类来创建依赖，比如我们创建一个 ```StockStrategyFactory``` 用来生产 ```StockStrategy```：

```
public class StockStrategyFactory {

    public static StockStrategy getStockStrategy() {
        return new StockStrategyImpl();
    }
}
```

然后我们再修改下客户端代码

```
    public Client() {
        this.strategy = StockStrategyFactory.getStockStrategy();
    }
```

这时如果需要更换策略只需要修改```StockStrategyFactory```一个地方就可以了。但是这样就完美了吗？不是的，因为获取某个接口的实现类这种需求太多了，我们不可能针对每种情况都写一个```Factory```，这时我们就特别需要一个通用的```Factory```。

```
public class BeanFactory {

    public static <T> T getBean(Class<T> requiredType) {
        // 具体实现省略
        return null;
    }
}
```


## 总结

本篇文章我们从一个简单的例子中了解到客户端与具体实现之间耦合会导致各种问题，而使用 IoC 的思想可以帮助我们解决这些问题，我们最后实现的 ```BeanFactory``` 其实就可以看做是一个简易版的 IoC 容器了。到这里我们其实可能还是不太清楚到底什么是 IoC，下篇文章我们继续学习。




---

![](http://ww3.sinaimg.cn/large/0082lgKxgy1gdhu6adriej31hb0hqace.jpg)