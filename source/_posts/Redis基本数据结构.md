---
title: Redis基本数据结构
categories:
  - Redis
tags:
  - Redis
date: 2019-03-05 10:09:40
---

Redis中包含string、hash、list、set、zset五种基本数据结构，本篇文章我们一起来学习一些这五种数据结构的基本用法和使用场景。<!-- more -->

## string

string 是 Redis 中最基本的数据结构，字符串类型的值实际可以是字符串(简单的字符串、复杂的字符串(JSON、XML))、数字(整数、浮点数)，甚至是二进制(图片、音频、视频)，但是值最大不能超过512MB。

### 基本用法

#### 设置值

```
set key value [ex seconds] [px milliseconds] [nx|xx]
```

- ex seconds : 设置秒级过期时间
- px milliseconds ： 设置毫秒级过期时间
- nx : 键必须不存在才能设置成功，用于新增
- xx : 键必须存在才能设置成功，用于更新

#### 获取值

```
get key
```

#### 批量设置值

```
mset key value [key value ...]
```

#### 批量获取值

```
mget key [key ...]
```

#### 计数

```
incr key
```


### 使用场景

#### 缓存

缓存是非常经典的使用场景，客户端获取数据时首先去缓存中获取，获取不到再到存储层去获取，在存储层获取到的数据再放到缓存中，下次就可以直接去缓存中获取了。

#### 计数

许多应用都会使用 Redis 作为基础计数工具，它可以实现快速计数，同时数据可以异步落地到其他数据源。

#### 共享 Session

在单机Web应用中，Session 通常存储在应用本地内存，但是对于分布式应用，Session 需要存储到外部系统，实现 Session 在各个应用中共享。通常我们会把 Session 存放到 Redis 中，对 Session 进行集中管理。

#### 限速

对于服务的某些接口，可能需要对其限速，比如同一手机号每天最多发5次短信或者每秒钟最多访问多少次，对于这种场景，可以在redis中记录访问次数，如果次数超过限制则进行限速。



## hash

Redis 是以键值对的形式存储数据，如果值本身也是键值对的形式，那么这种值的结构称之为hash。

### 基本用法

#### 设置值

```
hset key field value
```

#### 获取值

```
hget key field
```

#### 删除 field

```
hdel key field [field ...]
```

#### 批量设置或获取 field-value

```
hmset key field value [field value ...]
hmget key field [field ...]
```

#### 获取所有的field-value

```
hgetall key
```

### 使用场景

#### 缓存

上面我们讲到string可以用来做缓存，对于结构化的数据(比如用户数据，包含id、name、age等字段)我们也可以使用 hash 来做缓存，那么这两种方式有什么优缺点呢？

- 将用户数据序列化成字符串，然后用一个键保存
  - 优点：编程简单，合理使用序列化可以提高内存的使用效率
  - 缺点：序列化和反序列化有一定开销；每次都需要存取全部数据
- 使用hash，每个用户属性使用一对 field-value
  - 优点：可以对单个或多个属性进行存取；如果使用合理可以减少内存的使用
  - 缺点：控制 hash 在 ziplist 和 hashtable 两种内部编码的转换，hashtable 会使用更多内存



## list

list用来存储有序的字符串，列表中的每个字符串称之为元素，一个列表最多可以存储 2^23-1 个元素，列表可以两端插入和弹出，因此可以充当栈和队列的角色。

### 基本用法

#### 插入操作

```
# 左端插入
lpush key value [value ...]


# 右端插入
rpush key value [value ...]
```

#### 弹出操作

```
# 左端弹出
lpop key

# 右端弹出
rpop key
```

#### 查找操作

```
# 获取指定范围内的元素列表
lrange key start end

# 获取指定索引的元素
lindex key index
```

#### 修改操作

```
lset key index value
```

#### 阻塞操作

```
# 左端阻塞弹出
blpop key [key ...] timeout

# 右侧阻塞弹出
brpop key [key ...] timeout
```


### 使用场景

#### 消息队列

通过Redis的 lpush + brpop (或者 rpush + blpop)命令组合即可实现阻塞队列。



## set

set也是用来保存多个字符串元素，它和list不同之处在于，set不允许有重复元素，另外set中的元素是无序的。一个set中最多可以存储 2^23-1 个元素。

### 基本用法

#### 添加元素

```
sadd key value [value ...]
```

#### 删除元素

```
srem key value [value ...]
```

#### 随机返回指定个数的元素

```
# count 默认为1，注意返回的元素不会被删除
srandmember key [count]
```

#### 随机弹出指定个数的元素

```
# count 默认为1，注意返回的元素会被删除
spop key [count]
```

#### 多个集合的交集

```
# 直接输出结果
sinter key [key ...]

# 将结果保存到 resultset中
sinterstore resultset key [key ...]
```

#### 多个集合的并集

```
# 直接输出结果
sunion key [key ...]

# 将结果保存到 resultset中
sunionstore resultset key [key ...]
```


#### 多个集合的差集

```
# 直接输出结果
sdiff key [key ...]

# 将结果保存到 resultset中
sdiffstore resultset key [key ...]
```

### 使用场景

#### 标签

set 比较典型的使用场景是标签，通过给用户打标签，就可以找到拥有相同标签的人，或者找到不同用户公共的标签。


## zset

zset是有序集合，和set一样内部的元素不可以重复，但是和set不同的是zset中的元素可以根据score排序。

### 基本用法

#### 添加元素

```
zadd key score member [score member ...]
```

#### 删除指定排名内的升序元素

```
zremrangebyrank key start end
```

#### 返回指定排名范围的元素

```
# 升序范围内的元素
zrange key start end [withscores]

# 降序范围内的元素
zrevrange key start end [withscores]
```

#### 增加成员的分数

```
zincrby key score member
```

#### 多个有序集合的交集

```
zinterstore resultzset keynums key [key ...] [weights weight [weight ...]] [aggregate sum|max|min]
```

#### 多个有序集合的并集

```
zunionstore resultzset keynums key [key ...] [weights weight [weight ...]] [aggregate sum|max|min]
```

#### 多个有序集合的差集

```
zdiffstore resultzset keynums key [key ...] [weights weight [weight ...]] [aggregate sum|max|min]
```

### 使用场景

#### 排行系统

zset的典型使用场景就是需要按照某种规则排序的场景，比如排行榜系统。
