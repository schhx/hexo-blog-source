---
title: RabbitMQ消息可靠性
categories:
  - RabbitMQ
tags:
  - RabbitMQ
date: 2019-03-24 08:39:58
---

生产者发送消息一直到消费者消费消息，中间经过很长链路，如何才能保证消息在传输过程中不丢失呢？<!-- more -->这需要生产者、消费者和RabbitMQ 服务器共同努力才有可能。

## 生产者

生产者在发送消息时可能会遇到 RabbitMQ 服务不可用、没有对应的 Exchange 或者消息路由不到任何一个队列等情况，为了保证消息不丢失，我们需要在发送消息时指定 mandatory（强制的） 和 immediate（立即的）两个参数。

### 如何保证消息到达服务器

生产者把消息发送出去后，怎么判断消息有没有到达服务器呢？如果不做特殊的配置，默认情况下发送消息的操作是不会返回任何信息给生产者的，为了解决这个问题 RabbitMQ 提供了两种解决方案。

#### 事务机制

RabbitMQ 客户端提供了三个与事务相关的方法：

- channel.txSelect：开启事务
- channel.txCommit：提交事务
- channel.txRollback：回滚事务

使用事务机制的示例代码如下：

```
try {
    channel.txSelect();
    channel.basicPublish(EXCHANGE, ROUTING_KEY, MessageProperties.PERSISTENT_TEXT_PLAIN, msg.getBytes("UTF-8"));
    channel.txCommit();
} catch (Exception e) {
    e.printStackTrace();
    channel.txRollback();
}
```

如果事务提交成功了，则消息一定到达了 RabbitMQ 中。这里需要注意，如果消息和队列是持久化的，只有在消息写入磁盘后事务才能提交成功。

事务机制确实能够解决生产者和 RabbitMQ 之间消息确认的问题，但是会严重影响 RabbitMQ 性能，所以如果对性能有要求，不建议使用这种方法。

#### 发送方确认机制

生产者可以通过 ```channel.confirmSelect();``` 将信道设置为 confirm 模式，一旦信道进入 confirm 模式，所有在该信道上面发布的消息都会被指派一个唯一的ID，一旦消息被投递成功，RabbitMQ 就会发送一个确认给生产者（包含消息的唯一ID），如果消息和队列是持久化的，那么确认消息会在消息写入磁盘后发出。

使用发送方确认机制的示例代码如下：

```
try {
    channel.confirmSelect();
    channel.basicPublish(EXCHANGE, ROUTING_KEY, MessageProperties.PERSISTENT_TEXT_PLAIN, msg.getBytes("UTF-8"));
    if (!channel.waitForConfirms()) {
        System.out.println("send message fail");
        // do something else
    }
} catch (Exception e) {
    e.printStackTrace();
}
```

上面这种方式是每发送一条消息就同步等待服务器确认，其实这种方式并不比使用事务机制的吞吐量高多少。为了提高吞吐量可以使用一下两种方法：

- 批量 confirm：每发送一批消息后，调用 channel.waitForConfirms() 方法等待确认。
- 异步 confirm：通过 channel.addConfirmListener 添加 ConfirmListener 来处理回调信息。


### 如何保证消息被路由到队列

通过事务机制或者发送方确认机制，我们可以保证消息能够到达 RabbitMQ 服务器，注意这里到达服务器是指消息到达 RabbitMQ 的交换器，如果交换器没有匹配的队列，那么消息也会丢失。为了解决这种情况，我们需要在发送消息时指定 mandatory（强制的） 和 immediate（立即的）两个参数。

#### mandatory 参数

当 mandatory 为 true 时，如果交换器无法根据自身的类型和路由键找到一个符合条件的队列，那么 RabbitMQ 会调用 Basic.Return 把消息返回给消费者。当 mandatory 为 false 时，出现上述情况，则消息直接丢弃。mandatory 的默认值是 false。

#### immediate 参数

当 immediate 参数为 true 时，如果交换器在将消息路由到队列时发现队列并不存在消费者，那么这条消息将不会存入队列中；当路由键匹配的所有队列都不存在消费者时，该消息会通过 Basic.Return 返还给消费者。immediate 的默认值是false。

注意，RabbitMQ 3.0 已经废弃了对 immediate 参数的支持，也就是 immediate 参数只能为 false，否则客户端会报异常。

#### 接收服务器返回的信息

当服务器把消息返还给消费者时，消费者要怎么才能接收到返还的消息呢？这时候可以通过 channel.addReturnListener 来添加 ReturnListener 的实现。

```
channel.addReturnListener(new ReturnListener() {
    @Override
    public void handleReturn(int replyCode, String replyText, String exchange, String routingKey, AMQP.BasicProperties properties, byte[] body) throws IOException {
        String message = new String(body);
        System.out.println(message);
    }
});
```

#### 备份交换器

当生产者不设置 mandatory 参数时，消息在未被路由的情况下会丢失，设置了 mandatory 参数需要添加 ReturnListener 使得客户端代码复杂化，如果既不想消息丢失又不想复杂化客户端代码，那么可以使用备份交换器（Alternate Exchange，AE）。

可以给某个交换器设置备份交换器，设置完成后，如果消息未被正确路由，则消息就会发送到备份交换器，然后由备份交换器路由到匹配的队列中去。

备份交换器和普通的交换器没有什么太大区别，但是为了防止路由到备份交换器的消息丢失，建议将备份交换器的类型设置为 fanout。

使用备份交换器时，需要注意一下几种特殊情况：

- 如果设置的备份交换器不存在，则客户端和 RabbitMQ 服务器都不会有异常出现，消息丢失。
- 如果备份交换器没有绑定任何队列，则客户端和 RabbitMQ 服务器都不会有异常出现，消息丢失。
- 如果备份交换器没有匹配任何队列，则客户端和 RabbitMQ 服务器都不会有异常出现，消息丢失。
- 如果备份交换器和 mandatory 参数一起使用，则 mandatory 参数无效。


## RabbitMQ 服务器

消息被路由到了队列后就一定不会丢失吗？答案是否定的，如果消息到达服务器后没有持久化，那么消息很可能在服务器异常情况（重启、关闭、宕机）下丢失。为了解决这个问题，RabbitMQ 提供了持久化的功能来减少因服务器问题造成的消息丢失问题。

### 交换器持久化

交换器的持久化是通过声明交换器时将 durable 参数设置为 true 实现的，如果交换器不设置为持久化，服务器重启后，交换器的元数据会丢失，不过消息不会丢失，只是不能将消息发送到这个交换器中了。对于一个长期使用的交换器来说，建议将其设置为持久化。

### 队列持久化

队列的持久化是通过声明队列时将 durable 参数设置为 true 实现的，如果队列不设置为持久化，服务器重启后，队列的元数据会丢失，队列中的消息也会丢失。

但是将队列设为持久化只能保证队列的元数据不丢失，并不能保证消息不丢失。为了保证消息不丢失，还需要设置消息持久化。

### 消息持久化

消息的持久化是通过发送消息时指定投递模式（BasicProperties.deliveryMode = 2，MessageProperties.PERSISTENT_TEXT_PLAIN 封装了这个属性）来实现的。

可以将所有的消息都设置为持久化，但这样会严重影响 RabbitMQ 的性能，对于可靠性要求不是那么高的消息可以不采用持久化来提高整体的吞吐量。

### 镜像队列

上面我们说到 RabbitMQ 提供了持久化的功能来减少因服务器问题造成的消息丢失问题，也就是说交换器、队列和消息都持久化了以后，也不能保证消息百分之百不丢失。

这是因为 RabbitMQ 并不会为每条消息都进行同步存盘（调用 fsync 方法），所以消息可能只是写到了系统缓存而不是物理磁盘之中，如果这时服务器发生了重启等情况，还没来得及落盘的消息将会丢失。

解决这个问题可以使用镜像队列机制，相当于一个队列多个副本，这样主节点挂掉，还可以到从节点获取消息，虽然这样也不能完全保证消息不丢失，但是可靠性会高很多。

另外一个方式是生产者使用事务机制或发送方确认机制。上文我们说到，如果消息和队列是持久化的，

- 使用事务机制，只有在消息写入磁盘后事务才能提交成功。
- 使用发送方确认机制，确认消息会在消息写入磁盘后发出。



## 消费者

为了保证消息从队列可靠地到达消费者，RabbitMQ 提供了消息确认机制，消费者在订阅队列时，可以指定 autoAck 参数：

- 当 autoAck 为 false 时，RabbitMQ 会等待消费者显式地回复确认信息后才会从内存（或磁盘）中删除消息。
- 当 autoAck 为 true 时，RabbitMQ 会自动把发出去的消息设置为确认，然后从内存删除，而不管消费者是否真正消费到了这些消息。

### 确认消息

当 autoAck 为 false 时，如果服务器一直没有收到消费者的确认信号，并且消费此消息的消费者已经断开连接，则服务器会安排此消息重新进入队列。

消费者成功消费完消息后，可以显式地调用 channel.basicAck 来告诉服务器消息已经成功消费，服务器可以删掉此消息。

如果消费者接到消息后想要明确拒绝消息的话可以使用 channel.basicReject 或者 channel.basicNack，两个方法的区别是 basicNack 支持批量拒绝。拒绝消息时可以指定消息是否需要重新进入队列，如果不需要重新进入队列，则服务器会将消息删除，否则消息会重新进入队列，以便发送给下一个订阅的消费者。



