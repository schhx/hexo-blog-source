---
title: Redis阻塞
categories:
  - Redis
tags:
  - Redis
date: 2019-03-16 08:30:17
---


Redis 是典型的单线程架构，所有的读写操作都是在一条主线程中完成，当 Redis 用于高并发场景时，如果主线程出现阻塞，哪怕是很短时间，对于我们的应用来说都是灾难。<!-- more -->

Redis 出现阻塞后，我们首先要获取慢查询，然后再分析原因。

我们可以通过 ```slowlog get {n}```来获取最近的 n 条慢查询，默认执行时间超过 10 ms 则认为是慢查询。

而导致阻塞的原因大致可以分为内在原因和外在原因。

### 内在原因

#### 使用大对象

大对象在读写或者迁移的时候耗时都会比较长，所以在应用中我们应该避免使用大对象，如果有大对象，我们可以把它拆分成多个小对象处理。

我们可以通过 ```redis-cli -h {ip} -p {port} --bigkeys``` 命令来获取 Redis 实例的大对象。

#### API 使用不合理

Redis 中很多操作的时间复杂度是 O(n)，对于这些操作我们使用时要十分小心，如果操作的对象又是大对象的话，耗时会显著增长，比如对一个有上万元素的 hash 执行 hgetall 操作。

##### keys 操作

有时我们需要在 Redis 中找出所有满足条件的 key，然后做处理，Redis 提供了一个简单暴力的指令 keys 用来列出所有满足特定正则字符串规则的 key。

这个指令使用非常简单，提供一个简单的正则字符串即可，比如 ```keys *``` 或者 ```keys abc*```。

但是这个指令有两个很明显的缺点：

- 没有 offset、limit 参数，一次性吐出所有满足条件的 key。
- keys 算法是遍历算法，复杂度是 O(n)，如果实例中有千万级以上的 key，这个指令就会导致 Redis 服务卡顿。

所以我们应该避免使用这个命令，而应该使用 [scan](https://redis.io/commands/scan) 命令。


#### CPU 饱和

单线程 Redis 在处理命令时只能使用一个 CPU，如果 Redis 把单核 CPU 打满，将会导致 Redis 不能处理更多的命令。

通过 top 命令很容易查看 Redis 进程的 CPU 使用率，当出现 CPU 饱和时，首先要判断 Redis 并发量是否已经达到极限，我们可以通过命令 ```redis-cli - h {ip} -p {port} --stat``` 获取 Redis 的使用情况：

- 如果看到 Redis 平均每秒处理请求数很高（比如 6 万 + ），说明 Redis 并发量已经达到极限，这时任何优化都很难再有效果，只能通过集群做水平扩展来分摊 OPS 压力。
- 如果 OPS 很小，只有几百几千，这种情况就需要注意了，有可能使用了高算法复杂度的命令或者过度内存优化。

#### 持久化阻塞

对于开启了持久化功能的 Redis 节点，持久化有可能会导致阻塞。

##### fork 阻塞

fork 操作发生在 RDB 和 AOF 重写时，如果 fork 操作耗时过长，必然会导致主线程阻塞。我们可以通过 ```info stats``` 命令获取到 latest_fork_usec 指标。

##### AOF 刷盘阻塞

当我们开启 AOF 持久化功能时， 一般情况下文件刷盘采用每秒一次，后台线程每秒对 AOF 文件执行 fsync 操作，当硬盘压力过大时，fsync 操作需要等待，直到写入完成。当主线程发现距离上一次 fsync 操作成功超过 2 秒，为了安全他会阻塞主线程直到 fsync 操作完成。


### 外在原因

#### CPU 竞争

CPU 竞争是指 Redis 和其他服务部署在同一台物理机上，当其他进程过度消耗 CPU 时，将严重影响 Redis 的吞吐量。

#### 内存交换

内存交换（swap）对 Redis 来说是致命的，因为 Redis 保证高性能的一个重要前提是所有数据存储在内存中，如果操作系统把 Redis 使用的部分内存换出到硬盘，将会导致 Redis 性能急剧下降。

#### 网络问题

网络问题经常是引起 Redis 阻塞的原因，当网络环境不好或者 Redis 所在机器网卡被打满等都会造成 Redis 阻塞。



## 参考文档


- 《Redis开发与运维》
- [Redis latency](https://redis.io/topics/latency) 
