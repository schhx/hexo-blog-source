---
title: Java静态代理与动态代理
categories:
  - Java
tags:
  - Java
date: 2018-10-15 22:15:04
---

代理模式是非常常用的设计模式，在我们自己的业务代码和框架中都有广泛的应用，比如 Spring AOP 就是使用代理模式实现的。<!-- more -->代理可以分为静态代理和动态代理，而动态代理根据实现的方式不同又分为 JDK 动态代理和 Cglib 动态代理。

## 静态代理

### 基于接口的静态代理

基于接口的静态代理是最常用的静态代理，一般情况下我们说的静态代理就是指的基于接口的静态代理。

这种实现方式下，首先需要一个接口类和一个真实的实现类：

```
// 接口类
public interface ISubject {

    void sayHello();
}

// 真实实现类
public class Subject implements ISubject {

    @Override
    public void sayHello() {
        System.out.println("hello");
    }
}
```

代理类需要实现接口类，并且持有一个真实的实现类，代理类在实现方法时，把真正的业务操作委托给实现类，然后在委托前后加入自己的逻辑。

```
public class Proxy implements ISubject {

    private ISubject target;

    public Proxy(ISubject target) {
        this.target = target;
    }

    @Override
    public void sayHello() {
        System.out.println("proxy before");
        target.sayHello();
        System.out.println("proxy after");
    }
}
```

下面我们通过客户端来测试一下：

```
public class StaticClient {

    public static void main(String[] args) {
        ISubject proxy = new Proxy(new Subject());
        proxy.sayHello();
    }
}
```

输出结果如下，可以看出我们在不改变真实类的情况下，通过代理扩展来原来的功能。

```
proxy before
hello
proxy after
```


### 基于继承的静态代理

代理还可以通过继承真实类来实现，在实现方法时，把真正的业务操作委托给父类，然后在委托前后加入自己的逻辑。

```
public class Proxy2 extends Subject {

    @Override
    public void sayHello() {
        System.out.println("proxy2 before");
        super.sayHello();
        System.out.println("proxy2 after");
    }
}
```

同样我们来测试下：

```
public class StaticClient {

    public static void main(String[] args) {
        Subject proxy2 = new Proxy2();
        proxy2.sayHello();
    }
}
```

结果如下，可以看出达到了和基于接口的实现方式一样的效果。

```
proxy2 before
hello
proxy2 after
```

### 静态代理的缺点

静态代理适用于被代理类数量较少且确定的情况，如果被代理类数量较多，就会有许多重复的代码；或者被代理的对象在运行时才能确定情况，静态代理也处理不了。

## 动态代理

由于静态代理适用的情况较少，也不够灵活，在这样的背景下，出现了动态代理。动态代理能够在运行时动态地生成代理类，避免了静态代理的缺点，因此应用非常广泛。

动态代理现在有2种主流的实现方式：JDK 动态代理和 Cglib 动态代理。

### JDK 动态代理

JDK 动态代理是基于接口的动态代理，也就是必须要有接口才能使用 JDK 动态代理，这也是它的一个不足之处。

JDK 动态代理有一个核心的接口```InvocationHandler```和一个核心的类```Proxy```，```InvocationHandler```用来声明动态代理的通用逻辑，而```Proxy```则用来生成代理对象。示例代码如下：


```
public class JdkProxy implements InvocationHandler {

    private Object target;

    public JdkProxy(Object target) {
        this.target = target;
    }

    /**
     * @param proxy  代理类
     * @param method 目标类方法
     * @param args   方法参数
     * @return
     * @throws Throwable
     */
    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        System.out.println("jdk proxy before");
        Object result = method.invoke(target, args);
        System.out.println("jdk proxy after");
        return result;
    }
}
```

下面我们通过客户端来测试下：

```
public class JdkClient {

    public static void main(String[] args) {
        // 保存生成的动态代理类
        System.setProperty("sun.misc.ProxyGenerator.saveGeneratedFiles", "true");

        ISubject proxy = (ISubject) newProxyInstance(new Subject());
        proxy.sayHello();
    }

    public static Object newProxyInstance(Object target) {
        return Proxy.newProxyInstance(
                target.getClass().getClassLoader(),
                target.getClass().getInterfaces(),
                new JdkProxy(target)
        );
    }

}
```

结果如下：

```
jdk proxy before
hello
jdk proxy after
```

在客户端代码中加入```System.setProperty("sun.misc.ProxyGenerator.saveGeneratedFiles", "true")```可以保存生成的动态代理类，我们来看下动态生成的代理类：

```
public final class $Proxy0 extends Proxy implements ISubject {
    private static Method m1;
    private static Method m3;
    private static Method m2;
    private static Method m0;

    public $Proxy0(InvocationHandler var1) throws  {
        super(var1);
    }

    public final boolean equals(Object var1) throws  {
        try {
            return ((Boolean)super.h.invoke(this, m1, new Object[]{var1})).booleanValue();
        } catch (RuntimeException | Error var3) {
            throw var3;
        } catch (Throwable var4) {
            throw new UndeclaredThrowableException(var4);
        }
    }

    public final void sayHello() throws  {
        try {
            super.h.invoke(this, m3, (Object[])null);
        } catch (RuntimeException | Error var2) {
            throw var2;
        } catch (Throwable var3) {
            throw new UndeclaredThrowableException(var3);
        }
    }

    public final String toString() throws  {
        try {
            return (String)super.h.invoke(this, m2, (Object[])null);
        } catch (RuntimeException | Error var2) {
            throw var2;
        } catch (Throwable var3) {
            throw new UndeclaredThrowableException(var3);
        }
    }

    public final int hashCode() throws  {
        try {
            return ((Integer)super.h.invoke(this, m0, (Object[])null)).intValue();
        } catch (RuntimeException | Error var2) {
            throw var2;
        } catch (Throwable var3) {
            throw new UndeclaredThrowableException(var3);
        }
    }

    static {
        try {
            m1 = Class.forName("java.lang.Object").getMethod("equals", Class.forName("java.lang.Object"));
            m3 = Class.forName("org.schhx.javademo.proxy.ISubject").getMethod("sayHello");
            m2 = Class.forName("java.lang.Object").getMethod("toString");
            m0 = Class.forName("java.lang.Object").getMethod("hashCode");
        } catch (NoSuchMethodException var2) {
            throw new NoSuchMethodError(var2.getMessage());
        } catch (ClassNotFoundException var3) {
            throw new NoClassDefFoundError(var3.getMessage());
        }
    }
}
```

从代码中我们可以看出，$Proxy0 实现了 ISubject 接口，$Proxy0 调用 sayHello 方法时，把请求委托给了 InvocationHandler的invoke方法。



### Cglib 动态代理

由于 JDK 动态代理要求被代理类必须实现接口，因此一些没有实现接口的类想要实现动态代理就只能采用其他方法，比如采用 Cglib 动态代理。 Cglib 动态代理是基于继承的动态代理， Cglib 动态代理同样有一个核心的接口```MethodInterceptor```和一个核心的类```Enhancer```，```MethodInterceptor ```用来声明动态代理的通用逻辑，而```Enhancer ```则用来生成代理对象。示例代码如下：

```
public class CglibProxy implements MethodInterceptor {

    /**
     * @param obj    代理类
     * @param method 目标类方法
     * @param args   方法参数
     * @param proxy  方法代理
     * @return
     * @throws Throwable
     */
    @Override
    public Object intercept(Object obj, Method method, Object[] args, MethodProxy proxy) throws Throwable {
        System.out.println("cglib proxy before");
        // 调用代理类父类方法，即目标类方法
        // 注意不要使用 proxy.invoke(obj, args)
        Object result = proxy.invokeSuper(obj, args);
        System.out.println("cglib proxy after");
        return result;
    }
}
```

同样使用客户端测试下：

```
public class CglibClient {

    public static void main(String[] args) {
        // 保存生成的动态代理类
        System.setProperty(DebuggingClassWriter.DEBUG_LOCATION_PROPERTY, ".//");

        Subject proxy = (Subject) newProxyInstance(Subject.class);
        proxy.sayHello();


    }

    public static Object newProxyInstance(Class superclass) {
        //工具类
        Enhancer en = new Enhancer();
        //设置父类
        en.setSuperclass(superclass);
        //设置回调函数
        en.setCallback(new CglibProxy());
        //创建子类对象代理
        return en.create();
    }
}
```

结果如下：

```
cglib proxy before
hello
cglib proxy after
```

同样我们来看下 Cglib 动态生成的代理类，由于代理类比较复杂，我们只看其中比较关键的部分：

```
public class Subject$$EnhancerByCGLIB$$792079f5 extends Subject implements Factory {
    public final void sayHello() {
        MethodInterceptor var10000 = this.CGLIB$CALLBACK_0;
        if (this.CGLIB$CALLBACK_0 == null) {
            CGLIB$BIND_CALLBACKS(this);
            var10000 = this.CGLIB$CALLBACK_0;
        }

        if (var10000 != null) {
            var10000.intercept(this, CGLIB$sayHello$0$Method, CGLIB$emptyArgs, CGLIB$sayHello$0$Proxy);
        } else {
            super.sayHello();
        }
    }
}
```

可以看出代理类继承了被代理的类 Subject，在执行方法时，把请求委托给了MethodInterceptor执行。

## 总结

- 静态代理实现较简单，只要代理对象对目标对象进行包装，即可实现增强功能，但静态代理只能为一个目标对象服务，如果目标对象过多，则会产生很多代理类。
- JDK 动态代理要求被代理类必须实现一个或多个接口，是基于接口来实现动态代理，因此它不能代理 private 方法。
- Cglib 代理无需实现接口，是基于继承来实现动态代理，因此它不能代理 final 方法和 private 方法。

## [示例代码地址](https://github.com/schhx/java-demo/tree/master/src/main/java/org/schhx/javademo/proxy)
