---
title: Spring学习笔记--BeanFactory注册BeanDefinition
categories:
  - Spring学习笔记
tags:
  - Spring学习笔记
date: 2020-06-11 09:29:53
---

我们在 [Spring学习笔记--BeanFactory和ApplicationContext基本使用方法](https://mp.weixin.qq.com/s/TEtP9CCgZKIbiZj0pc2mfQ) 这篇文章中体验了如何使用 BeanFactory，<!-- more -->我现在把代码复制过来再一起回顾下。

```
public class BeanFactoryDemo {

    public static void main(String[] args) {
        // 读取配置文件
        BeanFactory beanFactory = new DefaultListableBeanFactory();
        BeanDefinitionReader reader = new XmlBeanDefinitionReader((BeanDefinitionRegistry) beanFactory);
        reader.loadBeanDefinitions(new ClassPathResource("spring.xml"));

        // 获取 Bean
        beanFactory.getBean(UserService.class).save();
        beanFactory.getBean(HelloWorld.class).hello("spring");
    }
}
```

从代码上看大致可以分为两个部分，第一部分是读取配置文件保存 BeanDefinition 到 BeanFactory，第二部分是从 BeanFactory 获取 Bean。

本篇文章我们就来大致分析下第一部分的源码，分析代码之前，我们先简单介绍下代码中的几个接口或类：

- ```BeanFactory```：基础的 IoC 容器，用来获取 Bean 实例。
- ```DefaultListableBeanFactory```：BeanFactory 接口的一个最终的实现类，它同时也实现了 BeanDefinitionRegistry 接口。
- ```BeanDefinitionReader```：负责从资源文件中读取 BeanDefinition。
- ```XmlBeanDefinitionReader```：BeanDefinitionReader 接口的一个实现类，负责从 XML 文件中读取 BeanDefinition。
- ```BeanDefinitionRegistry```：负责注入和持有 BeanDefinition。
- ```Resource```：Spring 体系内资源的统一抽象。
- ```ClassPathResource```：放置在 class path 下的资源。


## 读取配置文件

读取配置文件的前两行代码新建一个 DefaultListableBeanFactory 和 一个 XmlBeanDefinitionReader，这两行代码比较简单，我们就略过，直接看读取配置文件的部分。

### XmlBeanDefinitionReader.loadBeanDefinitions

BeanDefinitionReader 有四个重载的 loadBeanDefinitions 方法，我们查看了 XmlBeanDefinitionReader 的源码可以发现，这四个方法最终都会调用到内部的  ```loadBeanDefinitions(EncodedResource encodedResource)``` 方法，这个方法我简化下，源码如下：

```
	    /**
     * 从特定 XML 文件中读取 bean definitions
     */
    public int loadBeanDefinitions(EncodedResource encodedResource) throws BeanDefinitionStoreException {
        // 记录加载的资源文件，防止循环加载
        Set<EncodedResource> currentResources = this.resourcesCurrentlyBeingLoaded.get();
        if (currentResources == null) {
            currentResources = new HashSet<>(4);
            this.resourcesCurrentlyBeingLoaded.set(currentResources);
        }
        if (!currentResources.add(encodedResource)) {
            throw new BeanDefinitionStoreException(
                    "Detected cyclic loading of " + encodedResource + " - check your import definitions!");
        }

        try (InputStream inputStream = encodedResource.getResource().getInputStream()) {
            InputSource inputSource = new InputSource(inputStream);
            if (encodedResource.getEncoding() != null) {
                inputSource.setEncoding(encodedResource.getEncoding());
            }
            // 真正读取文件的入口
            return doLoadBeanDefinitions(inputSource, encodedResource.getResource());
        } catch (IOException ex) {
            throw new BeanDefinitionStoreException(
                    "IOException parsing XML document from " + encodedResource.getResource(), ex);
        } finally {
            currentResources.remove(encodedResource);
            if (currentResources.isEmpty()) {
                this.resourcesCurrentlyBeingLoaded.remove();
            }
        }
    }
```

这段代码主要做了两件事：

- 一是防止资源文件循环加载，比如有两个 XML 文件，两个文件相互 import，就会导致循环加载。
- 二是调用真正读取文件的入口 doLoadBeanDefinitions。

### XmlBeanDefinitionReader.doLoadBeanDefinitions

同样我把 doLoadBeanDefinitions 代码简化下，可以发现主要代码是两行，一行是把 XML 文件转化成 Document 对象，另一行是注册 BeanDefinition。第一行代码本次就不再深入分析了，我们主要看下第二行。

```
    /**
     * 真正读取 XML 文件的地方
     */
    protected int doLoadBeanDefinitions(InputSource inputSource, Resource resource)
            throws BeanDefinitionStoreException {

        try {
            // 将 XML 文件转化成 Document
            Document doc = doLoadDocument(inputSource, resource);
            // 注册 BeanDefinition
            int count = registerBeanDefinitions(doc, resource);
            if (logger.isDebugEnabled()) {
                logger.debug("Loaded " + count + " bean definitions from " + resource);
            }
            return count;
        }
        catch (BeanDefinitionStoreException ex) {
            throw ex;
        }
    }
```

### XmlBeanDefinitionReader. registerBeanDefinitions

在这段代码中比较重要的是构造 XmlReaderContext，XmlReaderContext 中持有了很多后面需要的对象，比如 Resource、XmlBeanDefinitionReader、NamespaceHandlerResolver等。

另外比较重要的是通过 BeanDefinitionDocumentReader 来读取并注册 BeanDefinitions。


```
    /**
     * 注册 Document 中包含的 bean definitions
     */
    public int registerBeanDefinitions(Document doc, Resource resource) throws BeanDefinitionStoreException {
        // 创建 BeanDefinitionDocumentReader
        BeanDefinitionDocumentReader documentReader = createBeanDefinitionDocumentReader();
        // 获取 BeanDefinitionRegistry 中当前持有的 BeanDefinition 数量
        int countBefore = getRegistry().getBeanDefinitionCount();
        // 1、构造 XmlReaderContext
        // 2、注册 BeanDefinitions
        documentReader.registerBeanDefinitions(doc, createReaderContext(resource));
        // 返回 BeanDefinitionRegistry 中新增的 BeanDefinition 数量
        return getRegistry().getBeanDefinitionCount() - countBefore;
    }
```

### BeanDefinitionDocumentReader.doRegisterBeanDefinitions

BeanDefinitionDocumentReader.registerBeanDefinitions 代码很简单，设置了 XmlReaderContext，然后直接调用 doRegisterBeanDefinitions。这里面首先是处理 profile 环境标识，通过这个属性我们可以在不同环境加载不同的 XML 文件。下面调用了三个方法，前置处理和后置处理都是空方法，留给子类去实现，这也相当于预留了一个扩展点。

```
    /**
     * 对给定的 <beans/> 根元素解析注册 bean definition
     */
    protected void doRegisterBeanDefinitions(Element root) {
        BeanDefinitionParserDelegate parent = this.delegate;
        this.delegate = createDelegate(getReaderContext(), root, parent);

        // 处理 profile
        if (this.delegate.isDefaultNamespace(root)) {
            String profileSpec = root.getAttribute(PROFILE_ATTRIBUTE);
            if (StringUtils.hasText(profileSpec)) {
                String[] specifiedProfiles = StringUtils.tokenizeToStringArray(
                        profileSpec, BeanDefinitionParserDelegate.MULTI_VALUE_ATTRIBUTE_DELIMITERS);
                if (!getReaderContext().getEnvironment().acceptsProfiles(specifiedProfiles)) {
                    if (logger.isDebugEnabled()) {
                        logger.debug("Skipped XML bean definition file due to specified profiles [" + profileSpec +
                                "] not matching: " + getReaderContext().getResource());
                    }
                    return;
                }
            }
        }

        // 前置处理
        preProcessXml(root);
        // 解析入口
        parseBeanDefinitions(root, this.delegate);
        // 后置处理
        postProcessXml(root);

        this.delegate = parent;
    }
```

### BeanDefinitionDocumentReader.parseBeanDefinitions

这段代码会判断 Element 的 namespace，如果 Element 的 namespace 是默认的 namespace（即 ```http://www.springframework.org/schema/beans```）的话就会最终调用 parseDefaultElement，否则的话调用 delegate.parseCustomElement。

parseDefaultElement 内部就是对 "import", "alias", "bean"， "beans" 四种标签进行解析。parseCustomElement就是对自定义标签进行解析，比如 ```<context:annotation-config/>```、```<context:component-scan"/>``` 这种标签就是在这里解析的，这里的具体源码我们下篇文章再分析。

```
    /**
     * 解析 Element
     * @param root
     * @param delegate
     */
    protected void parseBeanDefinitions(Element root, BeanDefinitionParserDelegate delegate) {
        if (delegate.isDefaultNamespace(root)) {
            NodeList nl = root.getChildNodes();
            for (int i = 0; i < nl.getLength(); i++) {
                Node node = nl.item(i);
                if (node instanceof Element) {
                    Element ele = (Element) node;
                    if (delegate.isDefaultNamespace(ele)) {
                        parseDefaultElement(ele, delegate);
                    }
                    else {
                        delegate.parseCustomElement(ele);
                    }
                }
            }
        }
        else {
            delegate.parseCustomElement(root);
        }
    }
    
    /**
     * 对 "import", "alias", "bean"， "beans" 进行解析
     */
    private void parseDefaultElement(Element ele, BeanDefinitionParserDelegate delegate) {
        if (delegate.nodeNameEquals(ele, IMPORT_ELEMENT)) {
            importBeanDefinitionResource(ele);
        } else if (delegate.nodeNameEquals(ele, ALIAS_ELEMENT)) {
            processAliasRegistration(ele);
        } else if (delegate.nodeNameEquals(ele, BEAN_ELEMENT)) {
            processBeanDefinition(ele, delegate);
        } else if (delegate.nodeNameEquals(ele, NESTED_BEANS_ELEMENT)) {
            doRegisterBeanDefinitions(ele);
        }
    }
```


### BeanDefinitionDocumentReader.processBeanDefinition

这里我们具体分析下对 ```<bean/>``` 标签的解析，首先通过 BeanDefinitionParserDelegate 将 ```<bean/>``` 转换成 BeanDefinitionHolder，然后通过 BeanDefinitionReaderUtils 将 BeanDefinition 注册到 BeanDefinitionRegistry 中，实际上是 DefaultListableBeanFactory 中。 


```
    /**
     * 解析 <bean/> 标签并注册
     */
    protected void processBeanDefinition(Element ele, BeanDefinitionParserDelegate delegate) {
        // 将 <bean/> 标签解析成 BeanDefinitionHolder
        BeanDefinitionHolder bdHolder = delegate.parseBeanDefinitionElement(ele);
        if (bdHolder != null) {
            bdHolder = delegate.decorateBeanDefinitionIfRequired(ele, bdHolder);
            try {
                // 注册 BeanDefinition
                BeanDefinitionReaderUtils.registerBeanDefinition(bdHolder, getReaderContext().getRegistry());
            }
            catch (BeanDefinitionStoreException ex) {
                getReaderContext().error("Failed to register bean definition with name '" +
                        bdHolder.getBeanName() + "'", ele, ex);
            }
            // 发送注册事件
            getReaderContext().fireComponentRegistered(new BeanComponentDefinition(bdHolder));
        }
    }
    
    /**
     * BeanDefinitionReaderUtils.registerBeanDefinition
     * 将 BeanDefinition 注册到 BeanDefinitionRegistry，实际上是 DefaultListableBeanFactory
     */
    public static void registerBeanDefinition(
            BeanDefinitionHolder definitionHolder, BeanDefinitionRegistry registry)
            throws BeanDefinitionStoreException {

        // 通过 primary name 注册
        String beanName = definitionHolder.getBeanName();
        registry.registerBeanDefinition(beanName, definitionHolder.getBeanDefinition());

        // 通过 alias 注册
        String[] aliases = definitionHolder.getAliases();
        if (aliases != null) {
            for (String alias : aliases) {
                registry.registerAlias(beanName, alias);
            }
        }
    }
```

## 总结

至此，我们大致分析了 Spring 读取 XML 文件并注册 BeanDefinition 的主要过程，当然还有很多细节没有分析到，比如 BeanDefinitionParserDelegate 将 ```<bean/>``` 转换成 BeanDefinitionHolder 的具体实现，但是我觉得读源码时首先是读主干逻辑，避免一开始就陷入细节中无法自拔，当了解主干逻辑后如果有需要可以有针对性地读某些细节。



---

![](http://ww3.sinaimg.cn/large/0082lgKxgy1gdhu6adriej31hb0hqace.jpg)