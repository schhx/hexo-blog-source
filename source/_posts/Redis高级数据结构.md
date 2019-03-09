---
title: Redis高级数据结构
categories:
  - Redis
tags:
  - Redis
date: 2019-03-06 18:16:58
---

本篇文章会简单介绍一些Redis的高级数据结构，包括Bitmaps、HyperLoglog、GEO。<!-- more -->

## Bitmaps

Bitmaps本身不是一种新的数据结构，它其实就是字符串，但是我们可以按位去操作它。

### 基本用法

#### 设置值

```
# 设置某一位的值为0或1
setbit key offset value
```

#### 获取值

```
getbit key offset
```

#### 获取指定范围内1的个数

```
bitcount key [start] [end]
```

- start：起始字节数
- end：结束字节数

#### Bitmaps间运算

```
bitop op result key [key ...]
```

- op：and、or、not xor
- 结果保存在result中

#### 计算Bitmaps中第一个 0|1 的偏移量

```
bitpos key 0|1 [start] [end]
```

- start：起始字节数
- end：结束字节数

### 使用场景

#### 记录用户行为

当用户行为只有两种状态时，比如签到（每天的状态只有签到和未签到两种状态），使用Bitmaps来记录用户行为可以大大减少内存的使用。

## HyperLogLog

HyperLogLog本身也不是一种新的数据结构（底层也是字符串），而是一种计数算法，HyperLogLog可以利用极小的内存空间完成独立总数的统计，但是HyperLogLog的计数并不完全准确，官方说法是有0.81%的误差。

### 基本用法

#### 添加元素

```
pfadd key element [element ...]
```

#### 计算独立元素数

```
pfcount key
```

#### 合并

```
pfmerge resultkey sourcekey [sourcekey ...]
```

### 使用场景

#### 独立计数

比如说我们要统计某个页面的访问用户数，但是要过滤掉相同的用户，也就是说一个用户访问页面多次，只统计一次，这个时候，我们就可以使用HyperLogLog，每次用户访问时就把userId放入HyperLogLog，最后就可以统计出访问的用户数。

### 原理

HyperLogLog的原理比较复杂，这里我们只是简单介绍。

用一句话来介绍就是：给定一系列的随机整数，我们记录下低位连续零位的最大长度 K，通过这个 K 值可以估算出随机数的数量 N。

```
N=2^K
```

如果 N 介于 2^K 和 2^(K+1) 之间，用这种方式估计的值都等于 2^K，这明显是不合理的。Redis采用16384个 BitKeeper，再进行加权估计，就可以得到一个比较准确的值。


## GEO

GEO（地理信息定位）底层使用的是zset。

### 基本用法

#### 增加地理位置

```
geoadd key longitude latitude member [longitude latitude member ...]
```

#### 获取地理位置

```
geopos key member [member ...]
```

#### 获取两个地理位置的距离

```
geodist key member1 member2 [unit]
```

- unit 表示单位，包含以下四种：m (米)、km (千米)、mi (英里)、ft (尺)

#### 获取指定位置范围内的地理信息位置集合

```
georadius key longitude latitude radius m|km|ft|mi

georadiusbymember key member radius m|km|ft|mi
```

#### 获取geohash

```
geohash key member [member ...]
```

### 使用场景

#### 附近的人

### 原理

业界比较通用的地理位置距离排序算法是 GeoHash 算法，Redis 也使用 GeoHash 算法。GeoHash 算法将二维的经纬度数据映射到一维的整数，这样所有的元素都将在挂载到一条线上，距离靠近的二维坐标映射到一维后的点之间距离也会很接近。当我们想要计算「附近的人时」，首先将目标位置映射到这条线上，然后在这个一维的线上获取附近的点就行了。

### 注意事项

在一个地图应用中，车的数据、餐馆的数据、人的数据可能会有百万千万条，如果使用 Redis 的 Geo 数据结构，它们将全部放在一个 zset 集合中。在 Redis 的集群环境中，集合可能会从一个节点迁移到另一个节点，如果单个 key 的数据过大，会对集群的迁移工作造成较大的影响，在集群环境中单个 key 对应的数据量不宜超过 1M，否则会导致集群迁移出现卡顿现象，影响线上服务的正常运行。

所以，这里建议 Geo 的数据使用单独的 Redis 实例部署，不使用集群环境。

如果数据量过亿甚至更大，就需要对 Geo 数据进行拆分，按国家拆分、按省拆分，按市拆分，在人口特大城市甚至可以按区拆分。这样就可以显著降低单个 zset 集合的大小。



