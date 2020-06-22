---
title: SpringBoot入门教程--创建新项目
categories:
  - SpringBoot入门教程
tags:
  - SpringBoot入门教程
date: 2020-04-04 11:21:08
---

## 新建项目

新建项目的方式有很多种，这里我推荐大家使用 [IDEA](https://www.jetbrains.com/idea/) 来新建 SpringBoot 项目。<!-- more -->

### 第一步
打开 IDEA，选择 File => New => Project... 然后看到如下画面，选择 Spring Initializr，右侧选择 Initializr Service URL，这里我们使用默认值。

![](http://ww3.sinaimg.cn/large/0082lgKxgy1gdhkozxmh4j31fs0u07ht.jpg)

### 第二步
点击 Next，就可以看到填写项目信息的界面，在这里可以填写项目的 Group 和 Artifact，项目 Type 可以根据需要选择 Maven 或者 Gradle，Packaging的方式可以选择 Jar 或者 War，推荐使用 Jar。

![](http://ww3.sinaimg.cn/large/0082lgKxgy1gdhkqpa1m4j31fs0u0qfw.jpg)

### 第三步
点击 Next，可以看到整个新建项目最重要的一步，选择 SpringBoot 的版本以及项目依赖的各种组件，这里除了 SpringBoot 的依赖以外，还有 SpringCloud 的各种依赖。由于我们是体验 SpringBoot，所以只选择了最基本的组件。

![](http://ww3.sinaimg.cn/large/0082lgKxgy1gdht8lkkstj31fs0u0h00.jpg)

### 第四步
点击 Next，填写项目名称和项目路径即可完成整个项目的创建。

![](http://ww3.sinaimg.cn/large/0082lgKxgy1gdhkw8qu6zj31fs0u0qe9.jpg)

## 修改项目

- 项目新建完成后，我们先来观察下整个项目的结构。首先我们会看到一个名为 SpringBootSimpleTutorialApplication 的 Java 类，这个类就是我们整个项目的启动类。

- resources 文件夹下有一个名为 application.properties 的文件，这个文件用来保存我们项目的各种配置，这里推荐大家把 application.properties 后缀改成 yml，使用 application.yml 来保存项目配置，因为yml格式更简洁。

- 删除一些不用的文件(不删也无所谓)，整个项目会如下图所示。

![](http://ww3.sinaimg.cn/large/0082lgKxgy1gdhlasi85vj31c00u0jyu.jpg)

- 点击 Run => Edit Configurations (或者从快捷菜单栏进入) 做如下配置，当我们改动代码时可以热加载。

![](http://ww3.sinaimg.cn/large/0082lgKxgy1gdhll0jk1dj31d70u0wrd.jpg)


## 测试

- 新建 controller 文件夹，并新建一个 HelloworldController 类。

![](http://ww3.sinaimg.cn/large/0082lgKxgy1gdhlna9zg1j31c00u0132.jpg)

- 因为我们还没有配置数据库，所以注释掉 ```pom.xml``` 中 ```spring-boot-starter-jdbc``` 的依赖，防止启动报错。

- 启动项目， 访问 http://localhost:8080/hello/schhx ，可以看到返回的结果 Hello, schhx !


---

![](http://ww3.sinaimg.cn/large/0082lgKxgy1gdhu6adriej31hb0hqace.jpg)



