---
title: Redis-Redis简单限流
date: 2019-03-10 23:16:50
categories: Redis
---

# 为什么需要限流

在大数据量高并发访问时, 经常会出现服务或接口面对大量请求而不可用的情况, 甚至引发连锁反映导致整个系统崩溃, 限流的技术手段可以很好的应对这种情况, 当请求达到一定的并发数或速率, 就进行等待, 排队, 降级, 拒绝服务等策略, 虽然会影响到用户的体验, 但是可以有效的保证系统不崩溃, 限流常见算法: 令牌桶, 漏斗, 计数器算法和时间窗口

<!-- more -->

# Redis 基于时间窗口实现的简单限流

```java
public Boolean redisLimitingTest() {
    StatefulRedisConnection<String, String> connection = redisClient.connect();
    RedisCommands<String, String> redisCommands = connection.sync();

    String key = String.format("LIMITING:%s:%s", "LPW001", "LOGIN");
    long currTime = DateTime.now().getMillis();

    // 移除 1 秒之前的历史数据
    redisCommands.zremrangebyscore(key, Range.create(0, (double) (currTime - 1000)));

    // 获取当前用户存在于时间窗口的数据的数量
    List<String> limitingDataList = redisCommands.zrange(key, 0, currTime);

    // 每秒请求不能超过 5 次请求
    if (limitingDataList.size() >= 5) {
        redisCommands.expire(key, 600);
        log.info("[简单限流测试] 超过限流阈值");
        return false;
    } else {
        String value = UUID.randomUUID().toString().replace("-", "");
        redisCommands.zadd(key, (double) currTime, value);
        redisCommands.expire(key, 600);
        return true;
    }
}
```

`注: 为了保证操作的原子性, 可以使用执行 Lua 脚本的方式来实现上面的功能`