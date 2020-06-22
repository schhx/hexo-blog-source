---
title: Spring学习笔记--BeanFactory和ApplicationContext基本使用方法
categories:
  - Spring学习笔记
tags:
  - Spring学习笔记
date: 2020-05-01 15:10:15
---

Spring 提供了两个 IoC 容器：一个基础的版本 ```BeanFactory```和一个高级的版本```ApplicationContext```。本篇文章我们就来一起学习下 ```BeanFactory``` 和 ```ApplicationContext``` 的基本使用方法。<!-- more -->注意本篇文章不是一个详细的使用教程，所以介绍的比较粗略。

> 基于 Spring 5.2.5.RELEASE

## 基础准备工作

为了演示 IoC 的基本使用方法，我们需要新建一个 Java 工程，并引入如下依赖：

```
        <dependency>
            <groupId>org.springframework</groupId>
            <artifactId>spring-context</artifactId>
            <version>5.2.5.RELEASE</version>
        </dependency>
```

创建三个个类 ```HelloWorld ```、```UserService``` 和 ```UserDao```：

```
public class HelloWorld {

    public void hello(String name) {
        System.out.println("hello " + name);
    }
}

public class UserService {

    private UserDao userDao;

    public void setUserDao(UserDao userDao) {
        this.userDao = userDao;
    }

    public void save() {
        userDao.save();
    }
}

public class UserDao {

    public void save() {
        System.out.println("save user");
    }
}
```

## BeanFactory + XML 配置文件

通过 XML 配置文件的方式管理依赖关系是 Spring 最传统的做法，也是最早支持的方式，虽然这种方式现在已经不推荐使用了，但是了解这种方式还是十分有必要。

我们首先新建一个 ```spring.xml``` 文件，并把 ```UserService``` 和 ```UserDao``` 之间的依赖关系配置到 ```spring.xml``` 文件中。

```
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
        https://www.springframework.org/schema/beans/spring-beans.xsd">

    <bean id="helloWorld" class="org.schhx.relearnspring.beans.HelloWorld"/>
    <bean id="userDao" class="org.schhx.relearnspring.beans.UserDao"/>
    <bean id="userService" class="org.schhx.relearnspring.beans.UserService">
        <property name="userDao" ref="userDao"/>
    </bean>

</beans>
```

由于 ```XmlBeanFactory``` 从 Spring 3.1 开始已经被废弃，并建议使用 ```DefaultListableBeanFactory``` + ```XmlBeanDefinitionReader```的方式来替代 ```XmlBeanFactory```，所以我们可以参考 ```XmlBeanFactory``` 的实现写下如下的测试代码：

```
public class BeanFactoryDemo {

    public static void main(String[] args) {
        BeanFactory beanFactory = new DefaultListableBeanFactory();
        BeanDefinitionReader reader = new XmlBeanDefinitionReader((BeanDefinitionRegistry) beanFactory);
        reader.loadBeanDefinitions(new ClassPathResource("spring.xml"));
        beanFactory.getBean(UserService.class).save();
        beanFactory.getBean(HelloWorld.class).hello("spring");
    }
}
```


## ApplicationContext + XML 配置文件

其实在平时我们并不经常使用 BeanFactory，而是使用它的高级版本 ApplicationContext，那我们就来看下如何使用 ApplicationContext。

```
public class ApplicationContextDemo {

    public static void main(String[] args) {
        ApplicationContext context = new ClassPathXmlApplicationContext("spring.xml");
        context.getBean(UserService.class).save();
        context.getBean(HelloWorld.class).hello("spring");
    }
}
```


## 通过 annotation-config 简化 XML 文件

当 Spring 管理的对象较少时，把所有的依赖关系配置到 XML 文件并没有什么问题，但是当 Spring 管理几十甚至几百个对象时，而它们之间又有比较复杂的依赖关系时，完全通过 XML 来配置简直是灾难。从 Spring 2.5 开始 Spring 支持通过 ```annotation-config``` 的方式来简化 XML 的配置，我们重新写一个配置文件 ```annotation-config.xml```，这时我们不必在 XML 文件中维护对象之间的依赖关系，只需要声明哪些对象需要 Spring 管理即可。

```
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:context="http://www.springframework.org/schema/context"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
        https://www.springframework.org/schema/beans/spring-beans.xsd
        http://www.springframework.org/schema/context
        https://www.springframework.org/schema/context/spring-context.xsd">

    <context:annotation-config/>

    <bean id="helloWorld" class="org.schhx.relearnspring.beans.HelloWorld"/>
    <bean id="userService" class="org.schhx.relearnspring.beans.UserService"/>
    <bean id="userDao" class="org.schhx.relearnspring.beans.UserDao"/>

</beans>
```

对象之间依赖关系通过 ```@Autowired``` 注解来维护，这时我们需要修改 ```UserService```代码如下：

```
public class UserService {

    @Autowired
    private UserDao userDao;

    public void setUserDao(UserDao userDao) {
        this.userDao = userDao;
    }

    public void save() {
        userDao.save();
    }
}
```

测试代码如下：

```
public class AnnotationConfigDemo {

    public static void main(String[] args) {
        ApplicationContext context = new ClassPathXmlApplicationContext("annotation-config.xml");
        context.getBean(UserService.class).save();
        context.getBean(HelloWorld.class).hello("spring");
    }
}
```

## 通过 component-scan 简化 XML 文件

通过 annotation-config 确实能够简化 XML 文件，但是我们还是要在 XML 中声明哪些对象需要 Spring 来管理，那有没有更简单的方式呢？其实是有的，从 Spring 2.5 开始我们可以在 XML 中配置 ```component-scan```，让 Spring 自动扫描识别哪些对象需要管理，我们重新编写一个配置文件 ```component-scan.xml```。

```
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:context="http://www.springframework.org/schema/context"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
        https://www.springframework.org/schema/beans/spring-beans.xsd
        http://www.springframework.org/schema/context
        https://www.springframework.org/schema/context/spring-context.xsd">

    <context:component-scan base-package="org.schhx.relearnspring"/>

</beans>
```

这时我们需要通过一些注解来标明哪些类需要 Spring 来管理。

```
@Component
public class HelloWorld {

    public void hello(String name) {
        System.out.println("hello " + name);
    }
}

@Service
public class UserService {

    @Autowired
    private UserDao userDao;

    public void setUserDao(UserDao userDao) {
        this.userDao = userDao;
    }

    public void save() {
        userDao.save();
    }
}

@Repository
public class UserDao {

    public void save() {
        System.out.println("save user");
    }
}
```

同样我们写一个测试代码：

```
public class ComponentScanDemo {

    public static void main(String[] args) {
        ApplicationContext context = new ClassPathXmlApplicationContext("component-scan.xml");
        context.getBean(UserService.class).save();
        context.getBean(HelloWorld.class).hello("spring");
    }
}
```

## 完全抛弃 XML 文件

通过 component-scan 我们几乎不用编写 XML 文件，通过注解就能够完全实现功能，那么我们可以完全抛弃 XML 文件吗？当然可以，我们这时需要使用 ```AnnotationConfigApplicationContext```。

```
public class AnnotationConfigApplicationContextDemo {

    public static void main(String[] args) {
        ApplicationContext context = new AnnotationConfigApplicationContext("org.schhx.relearnspring");
        context.getBean(UserService.class).save();
        context.getBean(HelloWorld.class).hello("spring");
    }
}
```

## 使用 Java Config

虽然通过 ```@Component``` 等注解可以标记某些类让 Spring 管理，但是对于一些通过 jar 包引入的类，我们没办法对其加注解，如果我们也想让 Spring 管理这些类，一种办法就是使用传统的 XML 配置文件的方式，另外一种更流行的办法是通过 Java Config 的方式来配置。

比如 ```HelloWorld ```、```UserService``` 和 ```UserDao``` 三个类都是外部引入的，我们把注解全部去掉：


```
public class HelloWorld {

    public void hello(String name) {
        System.out.println("hello " + name);
    }
}

public class UserService {

    private UserDao userDao;

    public void setUserDao(UserDao userDao) {
        this.userDao = userDao;
    }

    public void save() {
        userDao.save();
    }
}

public class UserDao {

    public void save() {
        System.out.println("save user");
    }
}
```

这时我们添加一个配置类 ```BeanConfig```：

```
@Configuration
public class BeanConfig {

    @Bean
    public HelloWorld helloWorld() {
        return new HelloWorld();
    }

    @Bean
    public UserDao userDao() {
        return new UserDao();
    }

    @Bean
    public UserService userService(UserDao userDao) {
        UserService userService = new UserService();
        userService.setUserDao(userDao);
        return userService;
    }
}
```

我们还是使用 AnnotationConfigApplicationContext 来测试下：


```
public class AnnotationConfigApplicationContextDemo {

    public static void main(String[] args) {
        ApplicationContext context = new AnnotationConfigApplicationContext("org.schhx.relearnspring");
        context.getBean(UserService.class).save();
        context.getBean(HelloWorld.class).hello("spring");
    }
}
```

## 总结

本篇文章带着大家回顾了下 Spring IoC 基本使用方式的变迁史，由于 Spring Boot 的流行，现在大家基本上是使用 **注解 + Java Config** 的方式，而不再使用 XML 配置文件的方式了，但是了解传统的实现方式有助于我们更加深入地理解 Spring，因为每一次使用方式的升级都是对以前的改进和增强，后续我们也会更加深入地学习 BeanFactory 和 ApplicationContext。








---

![](http://ww3.sinaimg.cn/large/0082lgKxgy1gdhu6adriej31hb0hqace.jpg)