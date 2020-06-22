---
title: Spring学习笔记--BeanFactory体系简介
categories:
  - Spring学习笔记
tags:
  - Spring学习笔记
date: 2020-06-11 09:27:49
---

BeanFactory 是 Spring 提供的基础 IoC 容器，ApplicationContext 也是基于 BeanFactory 来实现相关功能，所以本篇文章我们一起学习下 BeanFactory 体系比较重要的接口。

## BeanFactory 体系总览

BeanFactory 体系除了包含 BeanFactory 接口本身以外，还包括子接口、抽象实现类以及具体的实现类，下图可以看到 BeanFactory 体系的继承关系。


![](http://ww3.sinaimg.cn/large/0082lgKxgy1gehb2u1da1j31940sgaff.jpg)

## 重要接口

### BeanFactory

BeanFactory 是最基础的一个接口，它定义了一些实现 IoC 容器的必要方法，比如通过名称获取 Bean、通过类型获取 Bean、判断是否包含某个 Bean。

BeanFactory 有三个子接口：```HierarchicalBeanFactory```、```ListableBeanFactory``` 和 ```AutowireCapableBeanFactory```，这三个子接口分别对 BeanFactory 的某一方面进行了扩展。


### HierarchicalBeanFactory

从名称上看这是个具有层次性的 BeanFactory，它扩展了一个比较重要的方法 ```getParentBeanFactory``` 获取父 BeanFactory。


### ListableBeanFactory

从名称上看这是个可列表的 BeanFactory，BeanFactory 本身只提供了通过类型获取一个 Bean 的方法，而 ListableBeanFactory 扩展为可以通过类型获取多个同一类型的 Bean。

### AutowireCapableBeanFactory

我们知道被 BeanFactory 管理的 Bean 如果依赖其他的 Bean 的话，Spring 会自动实现注入，但是当某些外部 Bean 没有被 Spring 管理，也想注入一些被 Spring 管理的 Bean 怎么办呢？一种办法是让这些外部 Bean 被 Spring 管理，另外一种办法就是通过 AutowireCapableBeanFactory 的自动注入功能。AutowireCapableBeanFactory 通常用于 第三方框架与 Spring 整合，对于普通应用一般用不到。

### ConfigurableBeanFactory

ConfigurableBeanFactory 继承自 HierarchicalBeanFactory，提供了对 BeanFactory 的配置能力，大多数 BeanFactory 实现类都会实现这个接口。设计这个接口的主要目的是给 Spring 内部使用，外部服务最好是使用 BeanFactory 或者 ListableBeanFactory。

### ConfigurableListableBeanFactory

ConfigurableListableBeanFactory 算是 BeanFactory 体系里面的集大成者，它同时继承了 ListableBeanFactory、AutowireCapableBeanFactory 和 ConfigurableBeanFactory。和 ConfigurableBeanFactory 类似，ConfigurableListableBeanFactory 也是一个对内提供配置能力的接口。



---

![](http://ww3.sinaimg.cn/large/0082lgKxgy1gdhu6adriej31hb0hqace.jpg)