---
title: Spring学习笔记--IoC容器的主要职责
categories:
  - Spring学习笔记
tags:
  - Spring学习笔记
date: 2020-04-21 15:31:04
---


通过前面的学习，我们知道了 IoC 反转的是对象创建的过程，<!-- more --> 由客户端创建依赖的对象反转为外部创建对象，客户端直接使用创建好的对象，客户端得到依赖的方式有两种：依赖查找和依赖注入。

## 主要职责

IoC 容器如果要提供了上述功能的话，就必须完成以下功能：

#### 创建对象
既然客户端不负责创建依赖的对象，那这个工作只能由 IoC 容器来完成。


#### 提供获取对象的 API
如果 IoC 容器提供依赖查找的功能，那它必须提供获取对象的 API，否则客户端没办法通过依赖查找获取依赖的对象。


#### 管理对象之间的依赖绑定
如果 IoC 容器提供依赖注入的功能，那么 IoC 容器必须知道业务对象之间的依赖关系以及注入的方式，否则 IoC 容器将不能正确地将对象注入到客户端。

## 管理绑定关系

IoC 容器前两个职责相对来说比较简单，鉴于依赖注入是 IoC 容器的主要实现方式，第三个职责才是 IoC 容器最艰巨最重要的职责。IoC 容器本身是不知道对象之间的绑定关系，它必须通过某种方式获取绑定关系后才能管理绑定关系。通常来说 IoC 容器获取绑定关系的途径有以下几种：

#### 直接编码方式

这种方式是最基本的实现方式，比如在 Spring 中我们可以通过以下代码管理 ```UserService``` 和 ```UserDao``` 的依赖关系。

```
    GenericApplicationContext context = new GenericApplicationContext();
    context.registerBean("userDao", UserDao.class);
    context.registerBean("userService", UserService.class, context.getBean("userDao"));
```

#### 配置文件方式

通过编码的方式管理依赖关系虽然是最基本的方式，但是这种方式比较繁琐，当有大量依赖关系需要维护时简直是灾难。所以主流的 IoC 容器都会提供通过配置文件的方式管理依赖关系。配置文件不限定于特定的格式，比如 properties 文件、XML 文件等，只要 IoC 容器能够解析就可以了，但是最常用的还是 XML 文件，比如在 Spring 中我们经常通过以下方式管理绑定关系：

```
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
        https://www.springframework.org/schema/beans/spring-beans.xsd">

    <bean id="userDao" class="org.schhx.relearnspring.beans.UserDao"/>
    <bean id="userService" class="org.schhx.relearnspring.beans.UserService">
        <property name="userDao" ref="userDao"/>
    </bean>

</beans>
```

```
ApplicationContext context = new ClassPathXmlApplicationContext("spring.xml");
```

#### 注解方式

通过注解来管理依赖关系是目前非常流行的方式，因为这种方式相比较配置文件的方式更加简单易用，我们只需要在需要管理的类上添加相应的注解就能实现管理绑定关系。

```
@Repository
public class UserDao {

    public void save() {
        System.out.println("save user");
    }
}

@Service
public class UserService {

    @Autowired
    private UserDao userDao;

    public void save() {
        userDao.save();
    }
}

ApplicationContext context = new AnnotationConfigApplicationContext("org.schhx.relearnspring");
context.getBean(UserService.class).save();
```

## 总结

本篇文章我们介绍了 IoC 容器的主要职责以及管理绑定关系的主要方式，目前有多种比较流行的 IoC 容器，比如 Avalon、Google Guice、Spring 等，当然现在国内最流行的就是 Spring 了，在后续的文章中我们会逐步介绍 Spring 的使用方式。


## 参考文档

- 《Spring 揭秘》



---

![](http://ww3.sinaimg.cn/large/0082lgKxgy1gdhu6adriej31hb0hqace.jpg)