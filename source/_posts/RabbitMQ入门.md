---
title: RabbitMQ入门
categories:
  - RabbitMQ
tags:
  - RabbitMQ
date: 2019-03-17 15:11:48
---

消息队列中间件（Message Queue Middleware，简称MQ）是指利用高效可靠的消息传输机制进行与平台无关的数据交流。RabbitMQ 正是一个比较流行的消息队列中间件，<!-- more -->它采用 Erlang 语言编写，实现了 AMQP 协议。

MQ 在不同的场景下可以展现不同的作用，大致有以下几种作用

- 冗余存储
- 扩展性
- 解耦
- 削峰填谷
- 异步通讯

## 基本架构

RabbitMQ 整体上是一个生产者消费者模型，它的整体架构如下图所示：

![](http://www.itmind.net/assets/images/2016/RabbitMQ01.png)

下面我们几个比较重要的概念：

### Broker 服务节点

对于 RabbitMQ 来说，一个 RabbitMQ Broker 可以简单地看作一个 RabbitMQ 服务节点，或者 RabbitMQ 实例。

### 虚拟主机

一个虚拟主机持有一组交换机、队列和绑定。为什么需要多个虚拟主机呢？很简单， RabbitMQ 当中，用户只能在虚拟主机的粒度进行权限控制。 因此，如果需要禁止A组访问B组的交换机/队列/绑定，必须为A和B分别创建一个虚拟主机。每一个 RabbitMQ 服务器都有一个默认的虚拟主机“/”。


### Exchange 交换机

生产者发出的消息并不会直接到队列中去，而是先经过交换机，然后由交换机根据路由键投递到相应的队列。

RabbitMQ 中有四种交换机，不同类型的交换机有着不同的路由策略，在具体讲解这四种交换机类型之前，我们先来看另外两个概念：RoutingKey（路由键）和 BindingKey（绑定键），路由键由生产者发送消息时指定，绑定键由队列绑定到交换机时指定，消息会被路由到哪些队列上就是根据路由键和绑定键的关系确定。

- fanout：把所有发送到该交换机的消息路由到所有与该交换机绑定的队列中。
- direct：把消息路由到 RoutingKey 和 BindingKey 完全匹配的队列中。
- topic：支持 RoutingKey 和 BindingKey 模糊匹配，其中 RoutingKey 是准确的，BindingKey 可以存在两个特殊字符 “*” 和 “#” 用于模糊匹配，“*”表示匹配一个单词，“#”表示匹配零个或等多个单词。
- headers：这种交换机不依赖路由键，而是靠发送消息的 headers 属性进行匹配，这种交换机性能很差，而且也不实用。

### 队列

队列在 RabbitMQ 中用于存储消息，多个消费者可以订阅同一个队列，这时队列中的消息会被平均分摊给多个消费者处理。

## 基本用法

我们事先在 RabbitMQ 服务器上新建好虚拟主机、交换机和队列，所以不用在代码中再次声明。

### 直接使用 amqp-client

首先需要在代码中加入相关依赖

```
<dependency>
    <groupId>com.rabbitmq</groupId>
    <artifactId>amqp-client</artifactId>
    <version>5.4.3</version>
</dependency>
```

下面就是直接使用 amqp-client 的示例代码：

```
public class RabbitMQDemo {

    private static final String HOST = "xx";
    private static final int PORT = 5672;
    private static final String USERNAME = "admin";
    private static final String PASSWORD = "admin";
    private static final String VIRTUAL_HOST = "admin";
    private static final String EXCHANGE = "amq.topic";
    private static final String ROUTING_KEY = "hello";
    private static final String QUEUE = "hello";

    public static void main(String[] args) throws Exception {
        publisher();
        consumer();
    }

    public static void publisher() throws Exception {
        ConnectionFactory factory = new ConnectionFactory();
        factory.setHost(HOST);
        factory.setPort(PORT);
        factory.setUsername(USERNAME);
        factory.setPassword(PASSWORD);
        factory.setVirtualHost(VIRTUAL_HOST);

        // 创建连接
        Connection connection = factory.newConnection();
        // 创建信道
        Channel channel = connection.createChannel();

        String msg = "hello";
        channel.basicPublish(EXCHANGE, ROUTING_KEY, MessageProperties.PERSISTENT_TEXT_PLAIN, msg.getBytes("UTF-8"));
        channel.close();
        connection.close();
    }

    public static void consumer() throws Exception {
        ConnectionFactory factory = new ConnectionFactory();
        factory.setHost(HOST);
        factory.setPort(PORT);
        factory.setUsername(USERNAME);
        factory.setPassword(PASSWORD);
        factory.setVirtualHost(VIRTUAL_HOST);

        Connection connection = factory.newConnection();
        final Channel channel = connection.createChannel();
        // 设置客户端最多接收未被 ack 的消息的个数
        channel.basicQos(64);
        Consumer consumer = new DefaultConsumer(channel) {
            @Override
            public void handleDelivery(String consumerTag,
                                       Envelope envelope,
                                       AMQP.BasicProperties properties,
                                       byte[] body)
                    throws IOException {
                System.out.println(new String(body));
                try {
                    TimeUnit.SECONDS.sleep(1);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }
                channel.basicAck(envelope.getDeliveryTag(), false);
            }
        };
        channel.basicConsume(QUEUE, fales, consumer);

        TimeUnit.SECONDS.sleep(5);
        channel.close();
        connection.close();
    }
}

```


### 在 Spring Boot 中使用

Spring Boot 是现在非常流行的框架，在 Spring Boot 使用 RabbitMQ 也非常简单。

**添加依赖**

```
<dependency>
	<groupId>org.springframework.boot</groupId>
	<artifactId>spring-boot-starter-amqp</artifactId>
</dependency>
```

**然后在配置文件中配置 RabbitMQ 相关信息**

```
spring:
  rabbitmq:
    host: xx
    port: 5672
    username: admin
    password: admin123
    virtual-host: test
```

**生产者**

```
@Component
@Slf4j
public class RabbitMQSender {
    @Autowired
    private RabbitTemplate rabbitTemplate;

    public void send(String context) {
        log.info("send: {}", "hello " + context);
        this.rabbitTemplate.convertAndSend("amq.topic", "hello", context);
    }
}
```

**消费者**

```
@Component
@RabbitListener(queues = "hello")
@Slf4j
public class RabbitMQReceiver {

    @RabbitHandler
    public void process(String context) {
        log.info("receive: {}", context);
    }
}
```

**测试**

```
@SpringBootApplication
public class SpringBootRabbitmqApplication {

    public static void main(String[] args) {
        ConfigurableApplicationContext context = SpringApplication.run(SpringBootRabbitmqApplication.class, args);

        RabbitMQSender sender = context.getBean(RabbitMQSender.class);
        for (int i = 0; i < 100; i++) {
            sender.send(String.valueOf(i));
        }
    }

}
```