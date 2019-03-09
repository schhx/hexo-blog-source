---
title: Spring-AOP简介
categories:
  - Spring
tags:
  - Spring
date: 2018-10-21 08:43:12
---

Spring 有两大核心功能：IOC 和 AOP，今天我们就来简单介绍下 AOP 以及在 Spring 中的 AOP。<!-- more -->

## AOP

AOP(Aspect Oriented Programming，即面向切面的编程)，AOP 是一种开发理念，或者说是一种开发模式，通过 AOP，我们可以把一些非业务逻辑的代码，比如安全检查，监控等代码从业务方法中抽取出来，以非侵入的方式与原方法进行协同。这样可以使原方法更专注于业务逻辑，代码结构会更加清晰，便于维护。

AOP 并不是Java独有，更不是 Spring 独有，很多语言都有 AOP，AOP 有自己的标准，也有机构在维护这个标准，Spring AOP 目前也遵循相关标准。

### 为什么需要 AOP

AOP 是为了让通用逻辑和业务逻辑分离而出现的，在 AOP 出现以前，如果我们想在每个方法前加入权限校验、记录日志等通用逻辑，最原始的方法就是写一些公共方法，然后在每个需要做通用处理的方法上调用这些方法，这样有什么样的缺点呢？

1. 这些通用逻辑和业务逻辑没什么关系，写在方法内会污染业务逻辑；
2. 造成代码重复，如果需要增加一个通用逻辑或者去除一个通用逻辑，都需要在每个方法上手动修改同样的内容；添加新的方法也要再写一遍通用逻辑。

这个时候我们可能会想通过动态代理的方式来给方法添加通用逻辑，这样就能解决上面的问题，没错，Spring AOP 正是使用动态代理来实现AOP的。但是，动态代理并不是唯一的实现方式。

### AOP 术语

#### 连接点 - Joinpoint

连接点是指程序执行过程中的一些点，比如方法调用、方法执行、字段设置/获取、异常处理执行、类初始化、甚至是 for 循环中的某个点。理论上, 程序执行过程中的任何时点都可以作为作为织入点, 而所有这些执行时点都是 Jointpoint。在 Spring AOP 中，仅支持方法级别的连接点。

#### 切点 - Pointcut

连接点是可以织入的位置，但是我们的程序中并不是所有的连接点都需要织入逻辑，比如我们只想对特定的方法进行织入逻辑，这个时候就需要切点了，切点是用来筛选连接点的，告诉我们那些连接点需要织入逻辑。比如：切点表达式 ```@Pointcut("execution(* *.find*(..))")``` 就是用来筛选以 find 开头的方法。

#### 通知 - Advice 

通知即我们定义的横切逻辑，比如我们可以定义一个用于监控方法性能的通知，也可以定义一个安全检查的通知等。如果说切点解决了通知在哪里调用的问题，那么现在还需要考虑了一个问题，即通知在何时被调用？是在目标方法前被调用，还是在目标方法返回后被调用，还在两者兼备呢？Spring 帮我们解答了这个问题，Spring 中定义了以下几种通知类型：

- 前置通知（Before advice）- 在目标方便调用前执行通知
- 后置通知（After advice）- 在目标方法完成后执行通知
- 返回通知（After returning advice）- 在目标方法执行成功后，调用通知
- 异常通知（After throwing advice）- 在目标方法抛出异常后，执行通知
- 环绕通知（Around advice）- 在目标方法调用前后均可执行自定义逻辑

#### 切面 - Aspect

切面 Aspect 整合了切点和通知两个模块，切点解决了 where 问题，通知解决了 when 和 how 问题。切面把两者整合起来，就可以解决 对什么方法（where）在何时（when - 前置还是后置，或者环绕）执行什么样的横切逻辑（how）的三连发问题。比如在下面的代码中，我们整合切点和通知构成了一个切面。

```
/**
 * 切面
 */
@Aspect
@Slf4j
@Component
public class LogAspect {

    /**
     * 切点
     */
    @Pointcut("execution(public * org.schhx.springbootlearn.controller..*(..))")
    public void pointcut() {

    }

    /**
     * 前置通知
     *
     * @param joinPoint 可以获取目标对象的信息,如类名称,方法参数,方法名称等
     */
    @Before("pointcut()")
    public void before(JoinPoint joinPoint) {
        log.info("args: {}", joinPoint.getArgs());
    }
}
```

#### 织入 - Weaving

现在我们有了连接点、切点、通知，以及切面等，可谓万事俱备，但是还差了一股东风。这股东风是什么呢？没错，就是织入。所谓织入就是在切点的引导下，将通知逻辑插入到方法调用上，使得我们的通知逻辑在方法调用时得以执行。

## AspectJ

AOP的实现技术有多种，其中与Java无缝对接的是一种称为AspectJ的技术。对于AspectJ，我们只会进行简单的了解，从而为理解Spring中的AOP打下良好的基础(Spring AOP 与AspectJ 实现原理上并不完全一致，但功能上是相似的)，毕竟Spring中已实现AOP主要功能，开发中直接使用Spring中提供的AOP功能即可，除非我们想单独使用AspectJ的其他功能。

AspectJ是一个Java实现的AOP框架，它能够对Java代码进行AOP编译（一般在编译期进行），让Java代码具有AspectJ的AOP功能（当然需要特殊的编译器），可以这样说AspectJ是目前实现AOP框架中最成熟，功能最丰富的语言，更幸运的是，AspectJ与Java程序完全兼容，几乎是无缝关联，因此对于有Java编程基础的同学，上手和使用都非常容易。

AspectJ 的示例代码，这里我们就不再列出了，有兴趣的同学可以进一步阅读 [关于 Spring AOP (AspectJ) 你该知晓的一切](https://blog.csdn.net/javazejian/article/details/56267036)。




## Spring AOP


Spring AOP 与 ApectJ 的目的一致，都是为了统一处理横切业务，但与AspectJ不同的是，Spring AOP 并不尝试提供完整的AOP功能，Spring AOP 更注重的是与Spring IOC容器的结合，并结合该优势来解决横切业务的问题，因此在AOP的功能完善方面，相对来说AspectJ具有更大的优势，同时,Spring注意到AspectJ在AOP的实现方式上依赖于特殊编译器(ajc编译器)，因此Spring很机智回避了这点，转向采用动态代理技术的实现原理来构建Spring AOP的内部机制（动态织入），这是与AspectJ（静态织入）最根本的区别。在AspectJ 1.5后，引入@Aspect形式的注解风格的开发，Spring也非常快地跟进了这种方式，因此Spring 2.0后便使用了与AspectJ一样的注解。请注意，Spring 只是使用了与 AspectJ 5 一样的注解，但仍然没有使用 AspectJ 的编译器，底层依是动态代理技术的实现，因此并不依赖于 AspectJ 的编译器。

### 基于注解的 Spring AOP 的使用

#### 切点表达式

切点表达式包括三个部分：指示器、通配符和运算符，下面我们就来详细讲解一些这三个部分：

##### 通配符

- ```*``` : 用于匹配任意数量的字符

```
// 匹配任意返回值, service 包及其子包下任意 public 方法
@Pointcut("execution(public * org.schhx.service..*(..))")
```

- ```+``` : 用于匹配指定的类及其子类

```
// 匹配实现了UserDao接口的所有子类的方法
@Pointcut("within(org.schhx.dao.UserDao+)")
```

- ```..``` : 用于匹配任意数量的包及其子包或者参数

```
// 匹配任意返回值, service 包及其子包下任意 public 方法
@Pointcut("execution(public * org.schhx.service..*(..))")
```

##### 指示器

指示器有很多，这里我对一些常用的指示器我会给出例子，不常用的只是列出个名字。

**匹配方法**

- execution 是我们最常用的指示器，它的语法是

```
// scope ：方法作用域，如public,private,protect
// returnt-type：方法返回值类型
// fully-qualified-class-name：方法所在类的完全限定名称
// method-name(parameters) 方法声明
// throws-exception 抛出异常的声明
execution(<scope> <return-type> <fully-qualified-class-name><method-name(parameters)> <throws-exception>)

例如：@Pointcut("execution(public * org.schhx.service..*(..))")
```

**匹配注解**

- @target
- @args
- @within

```
//匹配使用了MarkerAnnotation注解的类(注意是类)
@Pointcut("@within(org.schhx.annotation.MarkerAnnotation)")
```

- @annotation

```
//匹配使用了MarkerAnnotation注解的方法(注意是方法)
@Pointcut("@annotation(org.schhx.annotation.MarkerAnnotation)")
```

**匹配包/类型**

- within

```
//匹配 UserService 类中所有方法
@Pointcut("within(org.schhx.service.UserService)")

//匹配 org.schhx.service 包及其子包中所有类中的所有方法
@Pointcut("within(org.schhx.service..*)")
```

**匹配对象**

- this
- bean
- target

**匹配参数**

- args


##### 运算符

包括```&&```、```||```和```!```三种，用来多个表达式进行运算。

```
@Pointcut("execution(public * org.schhx.service..*(..)) || @annotation(org.schhx.annotation.MarkerAnnotation)")
```

#### 通知

通过切点表达式我们知道了在何处织入逻辑，那么通知就是告诉 Spring 在何时织入何种逻辑，我们知道有五种通知，Spring 也对应提供了五种注解。

- @Before

前置通知会在执行方法逻辑前执行，前置通知可以添加一个可选的参数 JoinPoint。

```
	/**
	 * 前置通知
	 *
	 * @param joinPoint 可以获取目标对象的信息,如类名称,方法参数,方法名称等
	 */
	@Before("pointcut()")
	public void before(JoinPoint joinPoint) {
	    log.info("args: {}", joinPoint.getArgs());
	}
```


- @AfterReturning

后置通知会在方法执行完毕，执行返回操作后执行，后置通知有两个可选的参数，JoinPoint 和方法返回值 returnValue，当方法没有返回值时，returnValue 为null。

```
	/**
	 * 后置通知
	 *
	 * @param joinPoint 可以获取目标对象的信息,如类名称,方法参数,方法名称等
	 */
	@AfterReturning("pointcut()")
	public void afterReturning(JoinPoint joinPoint) {
	    log.info("我是后置通知");
	}
	
	/**
	 * 后置通知
	 *
	 * @param joinPoint 可以获取目标对象的信息,如类名称,方法参数,方法名称等
	 * @param returnValue 方法返回值，如果需要，必须手动指定
	 */
	@AfterReturning(value = "pointcut()", returning = "returnValue")
	public void afterReturning2(JoinPoint joinPoint, Object returnValue) {
	    log.info("returnValue: {}", returnValue);
	}
```


- @AfterThrowing

异常通知是在方法执行抛出异常后执行，异常通知有两个可选的参数，JoinPoint 和方法抛出的异常。

```
	/**
	 * 异常通知
	 *
	 * @param joinPoint 可以获取目标对象的信息,如类名称,方法参数,方法名称等
	 * @param e         方法抛出的异常，如果需要，必须手动指定
	 */
	@AfterThrowing(value = "pointcut()", throwing = "e")
	public void afterThrowing(JoinPoint joinPoint, Throwable e) {
	    log.error("异常通知", e);
	}
```


- @After

最终通知有点类似于finally代码块，无论什么情况下都会执行。

```
	/**
	 * 最终通知
	 *
	 * @param joinPoint 可以获取目标对象的信息,如类名称,方法参数,方法名称等
	 */
	@After("pointcut()")
	public void after(JoinPoint joinPoint) {
	    log.info("最终通知");
	}
```
    
    
- @Around

环绕通知是一个万能通知，包括前面四种通知的全部功能。

```
    /**
     * 环绕通知
     *
     * @param pjp 除了 JoinPoint 的全部功能以外，还可以执行被通知的方法
     * @return
     * @throws Throwable
     */
    @Around("pointcut()")
    public Object logAround(ProceedingJoinPoint pjp) throws Throwable {
        try {
            log.info("前置通知");
            Object response = pjp.proceed();
            log.info("后置通知");
            return response;
        } catch (Throwable e) {
            log.error("异常通知", e);
            throw e;
        }
    }
```

#### 切面

通过整合切点和通知，我们就可以得到一个完整的切面，下面是一个完整的例子。

```
/**
 * 切面
 */
@Aspect
@Slf4j
@Component
public class LogAspect {

    /**
     * 切点
     */
    @Pointcut("execution(public * org.schhx.springbootlearn.controller..*(..))")
    public void pointcut() {

    }

    /**
     * 前置通知
     *
     * @param joinPoint 可以获取目标对象的信息,如类名称,方法参数,方法名称等
     */
    @Before("pointcut()")
    public void before(JoinPoint joinPoint) {
        log.info("args: {}", joinPoint.getArgs());
    }

    /**
     * 后置通知
     *
     * @param joinPoint 可以获取目标对象的信息,如类名称,方法参数,方法名称等
     */
    @AfterReturning("pointcut()")
    public void afterReturning(JoinPoint joinPoint) {
        log.info("我是后置通知");
    }

    /**
     * 后置通知
     *
     * @param joinPoint   可以获取目标对象的信息,如类名称,方法参数,方法名称等
     * @param returnValue 方法返回值
     */
    @AfterReturning(value = "pointcut()", returning = "returnValue")
    public void afterReturning2(JoinPoint joinPoint, Object returnValue) {
        log.info("returnValue: {}", returnValue);
    }

    /**
     * 异常通知
     *
     * @param joinPoint 可以获取目标对象的信息,如类名称,方法参数,方法名称等
     * @param e         方法抛出的异常
     */
    @AfterThrowing(value = "pointcut()", throwing = "e")
    public void afterThrowing(JoinPoint joinPoint, Throwable e) {
        log.error("异常通知", e);
    }

    /**
     * 最终通知
     *
     * @param joinPoint 可以获取目标对象的信息,如类名称,方法参数,方法名称等
     */
    @After("pointcut()")
    public void after(JoinPoint joinPoint) {
        log.info("最终通知");
    }


    /**
     * 环绕通知
     *
     * @param pjp 除了 JoinPoint 的全部功能以外，还可以执行被通知的方法
     * @return
     * @throws Throwable
     */
    @Around("pointcut()")
    public Object logAround(ProceedingJoinPoint pjp) throws Throwable {
        try {
            log.info("前置通知");
            Object response = pjp.proceed();
            log.info("后置通知");
            return response;
        } catch (Throwable e) {
            log.error("异常通知", e);
            throw e;
        }
    }
}
```

### Spring AOP 的实现原理

上文我们已经说过 Spring AOP 是基于动态代理来实现 AOP 的，关于动态代理的知识，大家可以参考我的另一篇文章[Java静态代理与动态代理](https://schhx.github.io/2018/10/15/Java%E9%9D%99%E6%80%81%E4%BB%A3%E7%90%86%E4%B8%8E%E5%8A%A8%E6%80%81%E4%BB%A3%E7%90%86/)。

Spring 提供了一个后置处理器 BeanPostProcessor 接口，通过实现该接口，用户可在 bean 初始化前后做一些自定义操作。Spring AOP 实现了 BeanPostProcessor 接口，在 bean 初始化完成后，去判断 bean 是否需要织入 AOP 逻辑，如果需要的话，则动态生成 bean 的一个代理，然后把代理对象返回给 Spring 容器，这样我们从容器中取出对应的 bean，其实是代理后的 bean，执行 bean 的方法时，就会执行织入的逻辑。

## 总结

本篇文章介绍了什么是 AOP、AOP 的一些核心术语，另外简单介绍了 Spring AOP 的使用以及实现原理，Spring AOP 内容很多，它的实现原理也比较复杂，本篇文章没涉及很多细节的东西，只是希望能够让大家认识 AOP 以及怎么使用它。 

## 参考文档

- [关于 Spring AOP (AspectJ) 你该知晓的一切](https://blog.csdn.net/javazejian/article/details/56267036)
- [Spring AOP 源码分析系列文章导读](http://www.tianxiaobo.com/2018/06/17/Spring-AOP-%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90%E7%B3%BB%E5%88%97%E6%96%87%E7%AB%A0%E5%AF%BC%E8%AF%BB/)
- [一起来谈谈 Spring AOP](https://juejin.im/post/5aa7818af265da23844040c6)


