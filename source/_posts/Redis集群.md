---
title: Redis集群
categories:
  - Redis
tags:
  - Redis
date: 2019-03-13 19:26:02
---

Redis Sentinel 解决了高可用的问题，但是还有一些问题 Sentinel 不能解决，<!-- more -->比如：

- 主节点的写能力受到单机限制
- 主节点的存储能力受到单机限制

这些问题就必须靠集群来解决了。

## Redis Cluster

在大数据高并发场景下，单个 Redis 实例往往会显得捉襟见肘，比如单机 Redis 实例内存不宜过大，单机并发不足以支撑业务等，这时就必须依靠集群来分摊压力。Redis Cluster 是 Redis 官方提供的集群解决方案，它能够非常优雅地解决上面出现的问题。

### 数据分区

Redis 集群首先要考虑的问题是如何把数据均匀地分布在各个节点上，Redis Cluster 采用的是虚拟槽分区，所有的 key 根据 hash 函数映射到 0 ~ 16383 整数槽内，计算公式：```slot = CRC16(key) & 16383```，然后集群内的每一个节点负责维护一部分槽以及槽所映射的数据。

Redis 虚拟槽分区的的特点：

- 解耦数据和节点的关系，简化了节点扩容和缩容的难度
- 节点自身维护槽的映射关系，不需要客户端或者代理服务维护槽分区元数据
- 支持节点、槽、key 之间的映射查询，用于数据路由、在线伸缩等场景


### 集群伸缩

集群扩容时，首先将新的节点加入集群，然后把其他节点负责的一部分槽以及对应的数据迁移到新的节点。

集群缩容时，先把需要下线节点负责的槽和数据迁移到其他节点，然后再做下线操作。

#### 迁移槽和数据

Redis 迁移的单位是槽，Redis 一个槽一个槽进行迁移，当一个槽正在迁移时，这个槽就处于中间过渡状态。这个槽在源节点的状态为 migrating，在目标节点的状态为 importing，表示数据正在从源流向目标。

整个迁移过程大致如下：

1. 对目标节点发送命令，让目标节点准备导入 slot
2. 对源节点发送命令，让源节点准备导出 slot
3. 在源节点上获取 slot 下的 count 个 key
4. 批量迁移相关 key 的数据
5. 重复3、4步，直到 slot 的下数据迁移完毕
6. 通知集群内所有主节点，slot 已经分配给目标节点


### 故障转移

Redis Cluster 自身实现了高可用，每一个主节点都应该有至少一个从节点，当主节点故障时，从节点能够代替主节点继续对外服务。

#### 故障发现

Redis 集群节点采用 Gossip 协议来广播自己的状态以及自己对整个集群认知的改变，如果一个节点发现某个节点失联了，则把这个节点标记为主观下线（pfail, possible fail）状态，并将这个消息向集群内的其他节点进行广播，当集群内大多数节点都认为这个节点失联了，则把这个节点标记为客观下线（fail）状态。

#### 主从切换

当故障节点变为客观下线后，如果下线节点是主节点则需要在它的从节点中选择出一个来替换它，从而保证集群的高可用。

主从切换的大致流程如下：

1. 对从节点进行资格检查，如果从节点与主节点失联时间过长则失去替换主节点的资格。
2. 集群内其他主节点投票选举出最优从节点来替换故障主节点。
3. 从节点取消复制变成主节点，把主节点负责的槽分配给自己，向集群内其他主节点广播自己成为主节点



### 客户端使用

#### 请求路由

客户端会维护槽位和节点的映射关系表，当客户端请求命令时，首先根据 key 计算出槽位，再找到对应的节点，然后会向节点发送命令。

#### MOVED 重定向

我们知道槽位和节点的映射关系在扩容或缩容时会变化，那客户端怎么怎样才能感知这种变化呢？同样，Redis 节点在接收到命令时，也会先计算出 key 锁对应的槽，然后根据槽找到对应的节点，如果节点正是接收命令的节点，则处理命令，否则恢复 MOVED 重定向错误，通知客户端请求正确的节点，这时客户端就会刷新本地的映射关系表，然后再想正确的节点重试命令。

#### ASK 重定向

节点在迁移槽的过程中，槽位下的数据会有一部分在源节点，另一部分在目标节点，这时客户端在请求数据时流程又会有些不同：

1. 客户端根据 key 计算出节点，然后向节点发命令，如果节点上 key 存在则执行命令。
2. 如果 key 不存在，则可能存在目标节点上，这时源节点会回复 ASK 重定向错误。
3. 客户端发送 asking 命令到目标节点上打开客户端连接标示，再执行命令，如果 key 存在则执行命令，否则返回数据不存在。

#### MOVED 与 ASK 区别

MOVED 与 ASK 虽然都有重定向的功能，但是它们有本质的区别，ASK 说明槽位还未迁移完成，只是临时重定向，客户端不会更新槽位节点映射关系；而 MOVED 说明槽位已经迁移完成，需要永久重定向，客户端会更新槽位节点映射关系。




### Redis Cluster 功能限制

Redis Cluster 虽然解决了单机内存、并发等方面的不足，但是它相对于单机版也存在一些功能上的限制：

- key 批量操作支持有限，只支持同一个 slot 下 key 的批量操作
- key 事务支持有限，只支持同一个节点下 key 的事务
- 不支持多数据库空间，只能使用 db0
- 复制结构只支持一层，从节点只能复制主节点，不支持树状复制结构

#### hash_tag

在集群模式下，批量操作支持有限，为了让某些 key 分布在同一个 slot 下，我们可以使用 hash_tag。

如果 key 中含有大括号，则大括号中的内容称之为 hash_tag，如果 key 含有 hash_tag，则计算槽位时只使用 hash_tag，否则使用整个 key 来计算槽位，也就是说具有相同 hash_tag 的 key 一定会分布在同一个 slot 上，也会分布在同一个节点上，它常常用于 Redis IO 优化。


