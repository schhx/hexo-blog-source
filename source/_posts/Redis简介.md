---
title: Redis简介
categories:
  - Redis
tags:
  - Redis
date: 2019-03-05 09:10:06
---

[Redis](https://redis.io/) 是一种基于键值对(key-value)的NoSQL数据库，与很多键值对数据库不同的是，Redis支持多种数据结构，包括string、hash、list、set、zset五种基本数据结构，以及Bitmaps、HyperLogLog、GEO等多种高级数据结构。<!-- more -->


## Redis为什么这么快

正常情况下，Redis执行命令的速度非常快，官方给出的数字是读写性能10万/秒，总结起来原因大致可以归纳为以下三点：

- Redis的所有数据都是存放在内存的
- Redis是使用C语言实现的，一般来说，C语言实现的程序执行速度相对会更快
- Redis使用了单线程架构
  - 避免了线程切换和竞态产生的消耗
  - 使用非阻塞IO，Redis使用epoll作为I/O多路复用技术的实现，再加上Redis自身的事件处理模型将epoll中的连接、读写、关闭都转换为事件，不在网络I/O上浪费过多的时间。

## Redis可以做什么

#### 缓存

Redis最常用的场景就是构建缓存，合理使用缓存能够加快数据的访问速度，降低后端数据源的压力，但是使用时必须警惕缓存穿透、缓存雪崩等问题。

下图是常见的Redis + MySQL 架构：

![](Redis缓存.png)

#### 分布式锁

在分布式应用中，为了保证同一时刻只有一个用户能够访问某个资源，这时候必须使用分布式锁，分布式锁的实现方式有多种，借助Redis可以很容易地构建一个简易的分布式锁。

```
/**
 * 基于redis实现的简易分布式锁
 * 复杂的实现可参考 http://redis.io/topics/distlock
 */
@Component
public class RedisLock {

    private ScheduledExecutorService executorService = Executors.newScheduledThreadPool(2);

    @Autowired
    StringRedisTemplate redisTemplate;

    /**
     * redis setNx的集群排它锁
     *
     * @param key
     * @param second
     * @return
     * @see <a href="http://redis.io/commands/setnx">Redis Documentation: SETNX</a>
     */
    public boolean lock(String key, long second) {
        Boolean set = redisTemplate.opsForValue().setIfAbsent(key, System.currentTimeMillis() + second * 1000 + "");
        if (set) {
            redisTemplate.expire(key, second, TimeUnit.SECONDS);
        } else {
            String value = redisTemplate.opsForValue().getAndSet(key, System.currentTimeMillis() + second * 1000 + "");
            if (value != null && Long.parseLong(value) + 1 <= System.currentTimeMillis()) {
                return true;
            }
        }
        return set;
    }

    /**
     * 集群排它锁 这种实现方式更好
     *
     * @param key
     * @param value
     * @param timeout
     * @param timeUnit
     * @return
     * @see <a href="http://redis.io/commands/set">Redis Documentation: SET</a>
     */
    public boolean lock(final String key, final String value, long timeout, TimeUnit timeUnit) {
        redisTemplate.execute((RedisCallback<Object>) connection -> {
            connection.set(key.getBytes(), value.getBytes(), Expiration.from(timeout, timeUnit), RedisStringCommands.SetOption.SET_IF_ABSENT);
            return null;
        });
        String lockValue = redisTemplate.opsForValue().get(key);
        return value.equals(lockValue);
    }

    /**
     * 解锁
     *
     * @param key
     * @param value
     */
    public void unlock(String key, String value) {
        doUnlock(key, value);
    }

    /**
     * 延时解锁
     *
     * @param key
     * @param value
     * @param delayTime
     * @param unit
     */
    public void unlock(final String key, final String value, long delayTime, TimeUnit unit) {
        if (delayTime <= 0) {
            doUnlock(key, value);
        } else {
            executorService.schedule(() -> doUnlock(key, value), delayTime, unit);
        }

    }

    private void doUnlock(final String lockKey, final String resourceValue) {
        String value = redisTemplate.opsForValue().get(lockKey);
        if (resourceValue.equals(value)) {
            redisTemplate.delete(lockKey);
        }
    }
}
```

#### 排行榜系统

很多网站都有排行榜系统，比如阅读量排行榜，Redis提供了list和zset数据结构，合理使用这些数据结构可以很方便地构建各种排行榜系统。

#### 计数器应用

计数器在网站中也拥有很重要的作用，比如视频的播放量、网页的浏览量，为了保证数据的实时性，每一次播放和浏览都要做加 1 操作，如果并发量很大对于传统关系数据库的性能是一种挑战，Redis天然支持计数器功能且性能也非常好。

#### 社交网络

粉丝、共同好友/喜好、推送、下拉刷新等是社交网站的必备功能，由于社交网站访问量通常比较大，且传统的关系型数据库不太适合保存这种类型的数据，Redis提供的数据结构可以相对比较容易地实现这些功能，比如我们可以把粉丝、好友等信息放到zset中。


#### 消息队列系统

Redis提供了发布订阅功能和阻塞队列功能，可以用来构建简易的消息队列，虽然比不上专业的消息队列，但是在要求不高的场景中，还是有用武之地。


## 参考文档

- Redis开发与运维
- [Redis实战：如何构建类微博的亿级社交平台](http://i.dataguru.cn/article-9284-1.html)
