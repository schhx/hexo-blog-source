---
title: Spring学习笔记--解析XML自定义标签
categories:
  - 随笔
tags:
  - 随笔
date: 2020-06-11 09:39:33
---


我们在 [Spring学习笔记--BeanFactory注册BeanDefinition](https://mp.weixin.qq.com/s/jrgPhH6UbcAaI1mg5HdIew) 中没有展开 BeanDefinitionParserDelegate.parseCustomElement 这个方法，parseCustomElement 主要是对自定义标签进行解析，<!-- more -->比如对 ```<context:annotation-config/>```、```<context:component-scan/>``` 的解析，本篇文章我们就来分析下这块内容。

### BeanDefinitionParserDelegate.parseCustomElement

首先是根据 Element 获取 namespaceUri，对于```<context:annotation-config/>```、```<context:component-scan/>``` 获取到的 uri 是 ```http\://www.springframework.org/schema/context```;


```
    /**
     * 解析自定义标签
     */
    @Nullable
    public BeanDefinition parseCustomElement(Element ele) {
        return parseCustomElement(ele, null);
    }

    /**
     * 解析自定义标签
     */
    @Nullable
    public BeanDefinition parseCustomElement(Element ele, @Nullable BeanDefinition containingBd) {
        String namespaceUri = getNamespaceURI(ele);
        if (namespaceUri == null) {
            return null;
        }
        NamespaceHandler handler = this.readerContext.getNamespaceHandlerResolver().resolve(namespaceUri);
        if (handler == null) {
            error("Unable to locate Spring NamespaceHandler for XML schema namespace [" + namespaceUri + "]", ele);
            return null;
        }
        return handler.parse(ele, new ParserContext(this.readerContext, this, containingBd));
    }
        
```

第二步是根据 namespaceUri 获取 NamespaceHandler，这个里面具体的代码我们可以不分析，处理逻辑就是从 ```spring.handlers``` 文件中获取对应 NamespaceHandler，我们可以看下 ```spring-context``` 包下 ```spring.handlers``` 文件中的内容：

```
http\://www.springframework.org/schema/context=org.springframework.context.config.ContextNamespaceHandler
http\://www.springframework.org/schema/jee=org.springframework.ejb.config.JeeNamespaceHandler
http\://www.springframework.org/schema/lang=org.springframework.scripting.config.LangNamespaceHandler
http\://www.springframework.org/schema/task=org.springframework.scheduling.config.TaskNamespaceHandler
http\://www.springframework.org/schema/cache=org.springframework.cache.config.CacheNamespaceHandler
```

可以看出 ```http\://www.springframework.org/schema/context``` 对应的 NamespaceHandler 是 ```org.springframework.context.config.ContextNamespaceHandler```。

第三步是 NamespaceHandler 解析对应的标签。

### ContextNamespaceHandler

下面我们一起看下 ContextNamespaceHandler 的实现，可以看到 ContextNamespaceHandler 里面对每一种标签都注册了一种 BeanDefinitionParser，比如 ```annotation-config``` 对应的是 ```AnnotationConfigBeanDefinitionParser```，```component-scan``` 对应的是 ```ComponentScanBeanDefinitionParser```，ContextNamespaceHandler 在解析对应标签时最终是委托给对应的 BeanDefinitionParser 来解析的。

```
public class ContextNamespaceHandler extends NamespaceHandlerSupport {

	@Override
	public void init() {
		registerBeanDefinitionParser("property-placeholder", new PropertyPlaceholderBeanDefinitionParser());
		registerBeanDefinitionParser("property-override", new PropertyOverrideBeanDefinitionParser());
		registerBeanDefinitionParser("annotation-config", new AnnotationConfigBeanDefinitionParser());
		registerBeanDefinitionParser("component-scan", new ComponentScanBeanDefinitionParser());
		registerBeanDefinitionParser("load-time-weaver", new LoadTimeWeaverBeanDefinitionParser());
		registerBeanDefinitionParser("spring-configured", new SpringConfiguredBeanDefinitionParser());
		registerBeanDefinitionParser("mbean-export", new MBeanExportBeanDefinitionParser());
		registerBeanDefinitionParser("mbean-server", new MBeanServerBeanDefinitionParser());
	}

}
```

### AnnotationConfigBeanDefinitionParser

我们看下 AnnotationConfigBeanDefinitionParser 在解析 ```<context:annotation-config/>``` 到底做了什么事情，最重要的一句代码就是通过 AnnotationConfigUtils 注册相关的 BeanFactoryPostProcessor 或者 BeanPostProcessor。

```
/**
 * 解析 <context:annotation-config/>
 */
public class AnnotationConfigBeanDefinitionParser implements BeanDefinitionParser {

	@Override
	@Nullable
	public BeanDefinition parse(Element element, ParserContext parserContext) {
		Object source = parserContext.extractSource(element);

		// 注册相关的 BeanFactoryPostProcessor 或者 BeanPostProcessor
		Set<BeanDefinitionHolder> processorDefinitions =
				AnnotationConfigUtils.registerAnnotationConfigProcessors(parserContext.getRegistry(), source);

		// Register component for the surrounding <context:annotation-config> element.
		CompositeComponentDefinition compDefinition = new CompositeComponentDefinition(element.getTagName(), source);
		parserContext.pushContainingComponent(compDefinition);

		// Nest the concrete beans in the surrounding component.
		for (BeanDefinitionHolder processorDefinition : processorDefinitions) {
			parserContext.registerComponent(new BeanComponentDefinition(processorDefinition));
		}

		// Finally register the composite component.
		parserContext.popAndRegisterContainingComponent();

		return null;
	}

}
```

BeanFactoryPostProcessor 和 BeanPostProcessor 都是 Spring 提供的一个扩展点，BeanFactoryPostProcessor 主要用于容器启动阶段动态添加、删除、修改容器中的 BeanDefinition；BeanPostProcessor 主要用于 Bean 实例化时对 Bean 进行动态处理，关于 BeanFactoryPostProcessor 和 BeanPostProcessor 我们后面还会讲到。现在我们看下 AnnotationConfigUtils 中帮我们注册了哪些注解的处理器。

- ConfigurationClassPostProcessor 实现了 BeanFactoryPostProcessor，主要用于处理注解```@Configuration```。
- AutowiredAnnotationBeanPostProcessor 实现了 BeanPostProcessor，主要用于处理注解 ```@Autowired```、```@Value```。
- CommonAnnotationBeanPostProcessor 实现了 BeanPostProcessor，主要用于处理 JSR-250 标准注解 ```@PostConstruct```、```@PreDestroy```、```@Resource```等。
- EventListenerMethodProcessor 实现了 BeanFactoryPostProcessor，主要用于处理注解```@EventListener```。

### ComponentScanBeanDefinitionParser

ComponentScanBeanDefinitionParser 用来处理 ```<context:component-scan/>```，我们看下具体流程。

#### ComponentScanBeanDefinitionParser.parse

```
    public BeanDefinition parse(Element element, ParserContext parserContext) {
        // 1、获取 basePackage
        String basePackage = element.getAttribute(BASE_PACKAGE_ATTRIBUTE);
        basePackage = parserContext.getReaderContext().getEnvironment().resolvePlaceholders(basePackage);
        String[] basePackages = StringUtils.tokenizeToStringArray(basePackage,
                ConfigurableApplicationContext.CONFIG_LOCATION_DELIMITERS);

        // 2、注册 bean definitions
        // 2.1 构造 ClassPathBeanDefinitionScanner
        ClassPathBeanDefinitionScanner scanner = configureScanner(parserContext, element);
        // 2.2 扫描路径获取 BeanDefinitionHolder 并注册
        Set<BeanDefinitionHolder> beanDefinitions = scanner.doScan(basePackages);
        // 2.3 注册组件
        registerComponents(parserContext.getReaderContext(), beanDefinitions, element);

        return null;
    }
```

ComponentScanBeanDefinitionParser.parse 主要做了两件事情：

第一件是获取 basePackage，首先是从配置文件中获取原始的配置，如果原始配置是占位符的话则替换成对应环境配置的值，最后按分隔符转换成数组。

第二件是注册 bean definitions，首先是配置扫描器 ClassPathBeanDefinitionScanner，然后通过 scanner 扫描配置的路径得到 BeanDefinitionHolder，最后注册到容器内。

#### ComponentScanBeanDefinitionParser.configureScanner

configureScanner 的代码最主要的逻辑是 new 了一个 ClassPathBeanDefinitionScanner，然后内部最重要的逻辑是添加默认的 AnnotationTypeFilter：

- 添加 ```@Component``` 的 AnnotationTypeFilter，除了支持 ```@Component``` 本身以外，还支持以 ```@Component``` 为元注解的的注解，比如 ```@Repository```、```@Service```、```@Controller``` 等。
- 添加 ```@ManagedBean```、```@Named``` 的 AnnotationTypeFilter。

```
    /**
     * ClassPathBeanDefinitionScanner 构造方法
     */
    public ClassPathBeanDefinitionScanner(BeanDefinitionRegistry registry, boolean useDefaultFilters,
                                          Environment environment, @Nullable ResourceLoader resourceLoader) {

        Assert.notNull(registry, "BeanDefinitionRegistry must not be null");
        this.registry = registry;

        if (useDefaultFilters) {
            // 注册默认 AnnotationTypeFilter
            registerDefaultFilters();
        }
        setEnvironment(environment);
        setResourceLoader(resourceLoader);
    }

    /**
     * ClassPathBeanDefinitionScanner.registerDefaultFilters
     */
    protected void registerDefaultFilters() {
        // 支持 @Component 注解
        this.includeFilters.add(new AnnotationTypeFilter(Component.class));
        ClassLoader cl = ClassPathScanningCandidateComponentProvider.class.getClassLoader();
        try {
            this.includeFilters.add(new AnnotationTypeFilter(
                    ((Class<? extends Annotation>) ClassUtils.forName("javax.annotation.ManagedBean", cl)), false));
            logger.trace("JSR-250 'javax.annotation.ManagedBean' found and supported for component scanning");
        }
        catch (ClassNotFoundException ex) {
            // JSR-250 1.1 API (as included in Java EE 6) not available - simply skip.
        }
        try {
            this.includeFilters.add(new AnnotationTypeFilter(
                    ((Class<? extends Annotation>) ClassUtils.forName("javax.inject.Named", cl)), false));
            logger.trace("JSR-330 'javax.inject.Named' annotation found and supported for component scanning");
        }
        catch (ClassNotFoundException ex) {
            // JSR-330 API not available - simply skip.
        }
    }
```

#### ClassPathBeanDefinitionScanner.doScan

```
    protected Set<BeanDefinitionHolder> doScan(String... basePackages) {
        Assert.notEmpty(basePackages, "At least one base package must be specified");
        Set<BeanDefinitionHolder> beanDefinitions = new LinkedHashSet<>();
        for (String basePackage : basePackages) {
            // 扫描路径获取 BeanDefinition
            Set<BeanDefinition> candidates = findCandidateComponents(basePackage);
            for (BeanDefinition candidate : candidates) {
                ScopeMetadata scopeMetadata = this.scopeMetadataResolver.resolveScopeMetadata(candidate);
                candidate.setScope(scopeMetadata.getScopeName());
                String beanName = this.beanNameGenerator.generateBeanName(candidate, this.registry);
                if (candidate instanceof AbstractBeanDefinition) {
                    postProcessBeanDefinition((AbstractBeanDefinition) candidate, beanName);
                }
                if (candidate instanceof AnnotatedBeanDefinition) {
                    AnnotationConfigUtils.processCommonDefinitionAnnotations((AnnotatedBeanDefinition) candidate);
                }
                if (checkCandidate(beanName, candidate)) {
                    BeanDefinitionHolder definitionHolder = new BeanDefinitionHolder(candidate, beanName);
                    definitionHolder =
                            AnnotationConfigUtils.applyScopedProxyMode(scopeMetadata, definitionHolder, this.registry);
                    beanDefinitions.add(definitionHolder);
                    // 注册 BeanDefinition
                    registerBeanDefinition(definitionHolder, this.registry);
                }
            }
        }
        return beanDefinitions;
    }
```

#### ComponentScanBeanDefinitionParser.registerComponents

在注册组件的时候会默认注册 annotation config processors，即 ```<context:component-scan/>``` 默认包含了 ```<context:annotation-config/>``` 的全部功能，除非手动禁用。

```
    protected void registerComponents(
            XmlReaderContext readerContext, Set<BeanDefinitionHolder> beanDefinitions, Element element) {

        Object source = readerContext.extractSource(element);
        CompositeComponentDefinition compositeDef = new CompositeComponentDefinition(element.getTagName(), source);

        for (BeanDefinitionHolder beanDefHolder : beanDefinitions) {
            compositeDef.addNestedComponent(new BeanComponentDefinition(beanDefHolder));
        }

        // 默认注册 annotation config processors
        boolean annotationConfig = true;
        if (element.hasAttribute(ANNOTATION_CONFIG_ATTRIBUTE)) {
            annotationConfig = Boolean.parseBoolean(element.getAttribute(ANNOTATION_CONFIG_ATTRIBUTE));
        }
        if (annotationConfig) {
            Set<BeanDefinitionHolder> processorDefinitions =
                    AnnotationConfigUtils.registerAnnotationConfigProcessors(readerContext.getRegistry(), source);
            for (BeanDefinitionHolder processorDefinition : processorDefinitions) {
                compositeDef.addNestedComponent(new BeanComponentDefinition(processorDefinition));
            }
        }

        readerContext.fireComponentRegistered(compositeDef);
    }
```

### 总结

BeanFactory 支持解析自定义标签，比如 ```<context:annotation-config/>```、```<context:component-scan/>```，帮我们自动注入了许多 BeanFactoryPostProcessor 和 BeanPostProcessor，但是注意 BeanFactory 并没有应用这些 PostProcessor，想要它们起作用我们必须手动调用或者使用 ApplicationContext。




---

![](http://ww3.sinaimg.cn/large/0082lgKxgy1gdhu6adriej31hb0hqace.jpg)