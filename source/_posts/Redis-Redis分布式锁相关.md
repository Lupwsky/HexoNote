---
title: Redis-Redis分布式锁相关
date: 2019-02-26 23:02:42
categories: Redis
---

# 分布式锁可用的四个条件

* 互斥性, 同一时间只有一个客户端持有锁
* 无死锁, 即使有一个客户端在持有锁的期间崩溃而没有主动解锁, 也能保证后续其他客户端能加锁
* 容错性, 只要大部分节点正常运行, 客户端就可以自己加锁和解锁
* 标识性, 锁需要具备拥有者标识, 加锁和解锁必须是同一个客户端, 客户端不能解锁另一个客户端的锁

# 使用 setnx 和 expire 组合实现

使用 setnx 和 expire 组合实现加锁, 这是老版本的 Redis 时的方案, 问题: 在执行 setnx 后设置过期时间时客户端突然崩溃 (不是出现异常), 此时锁过期时间为永久有效, 其他客户端一直不能获取到锁, 就会出现死锁, 生产环境不要使用这种方式, 出现死锁肯定是不行, 同时锁没有标识, 其他的客户端可以解锁

```java
public boolean lock1(String lockKey, String lockValue, int lockTimeout) {
    try {
        StatefulRedisConnection<String, String> connection = redisClient.connect();
        RedisCommands<String, String> redisCommands = connection.sync();
        // 判断锁是否已被其他的客户端持有, 如果其他客户端有持有, 获取锁失败
        String lock = redisCommands.get(lockKey);
        if (lock != null) {
            return Boolean.FALSE;
        } else {
             // 在 String lock = redisCommands.get(lockKey) 在并发情况下可能获取到的值都是 null
             // 多个客户端会进入这里的加锁逻辑, 在这里再加一层判断
            if (!redisCommands.setnx(lockKey, lockValue)) {
                return Boolean.FALSE;
            } else {
                // 客户端在这里崩溃, 导致锁的过期时间为永久有效而出现死锁
                redisCommands.expire(lockKey, lockTimeout);
                return Boolean.TRUE;
            }
        }
    } catch (Exception e) {
        log.error("获取锁错误, error = {}", e);
        return Boolean.FALSE;
    }
}
```

<!-- more -->

# 使用 setnx 实现

任然使用 setnx, 但是将当前加锁时间作为锁的值, 每次获取锁前获取值比较时间来判断是否有客户端持有锁, 问题: 由于使用时间作比较, 有时间同步问题需要取解决

几个解决时间同步的方法:

* 如果时间从客户端获取, 客户端的服务器时间需要进行同步
* 使用一台服务器作为时间服务器
* 读取 Redis 服务器的时间

这种方案也是不推荐使用的, 主要是锁没有标识, 其他的客户端可以解锁, 例如:

* T1 在获取到锁后, 由于 GC 或者其他长时间操作超过锁过期时间
* 这个时候 T2 又获取到了锁开始执行任务, 接着 T1 执行完任务又解锁了
* 此时任意客户端如 T3 又可以获取锁并开始执行任务了

这样就可能造成程序上的异常

```java
public boolean lock2(String lockKey, int lockTimeout) {
    try {
        StatefulRedisConnection<String, String> connection = redisClient.connect();
        RedisCommands<String, String> redisCommands = connection.sync();

        // 当前时间
        List<String> currentTimeList = redisCommands.time();
        long currentTime = Long.valueOf(currentTimeList.get(0));

        // 假设同时有两个线程 T1, T2 同时在获取锁
        // 同时执行了 redisCommands.setnx(lockKey, String.valueOf(currentTime + lockTimeOut))
        // T1 设置成功, 进入了 true 分支, 返回 ture, 表示成功获取到锁, 锁过期时间为假设为 currentTime + 10s
        // T2 设置失败, 进入了 false 分支, 此时去获取当前锁设置的时间 lockTime = currentTime + 10s, 是大于当前 currentTime 的, 表示获取锁失败
        if (redisCommands.setnx(lockKey, String.valueOf(currentTime + lockTimeout))) {
            return Boolean.TRUE;
        } else {
            String value = redisCommands.get(lockKey);
            if (Objects.equals("", value)) {
                redisCommands.del(lockKey);
                return Boolean.FALSE;
            } else {
                long lockTime = Long.valueOf(value);
                if (currentTime <= lockTime) {
                    return Boolean.FALSE;
                } else {
                    redisCommands.set(lockKey, String.valueOf(currentTime + lockTimeout));
                    return Boolean.TRUE;
                }
            }
        }
    } catch (Exception e) {
        log.error("获取锁错误, error = {}", e);
        return Boolean.FALSE;
    }
}
```

# 使用 Redis 2.6 开始 set 支持添加时间参数实现

可以借助这个新特性来实现分布式锁, 这个方案是目前最优的分布式锁方案, 实现如下代码:

其中的 lockValue 设置为这个客户端唯一的 ID, 在解锁的时候通过着 ID 进行判断是否和加锁的是否是同一个 ID, 确定标识锁的唯一性,但是这种方式在集群的情况下有一点问题, 由于 Redis 集群数据同步为异步, 假设在Master节点获取到锁后未完成数据同步情况下 Master 节点挂掉了,此时在新的 Master 节点依然可以获取锁, 这样就会导致有两个客户端持有锁在执行任务 (解决方法是 RedLock), 如果业务需求能允许这样的情况这种情况也已经够用了

当然这种方案也是有由于某个持有锁的客户端出现 GC 或者其他长时间操作超过锁过期时间的情况的, 避免耗时操作或者适当延长超时时间, 例如:

* T1 获取锁后由于GC 或者其他长时间操作超过锁过期时间
* T2 获取到了锁
* T1 执行完成了任务, 根据唯一标识去释放了锁, 由于锁已经被 T2 获取, 且 ID 也是 T2 设置的, 由于 ID 不同, T1 释放锁失败, 不会释放 T2 的锁
* T3 开始获取锁, 由于 T2 还在持有锁, T3 获取锁失败, 等待 T2 释放锁后才能获取锁成功

```java
public boolean lock3(String lockKey, String lockValue, int lockTimeout) {
    try {
        StatefulRedisConnection<String, String> connection = redisClient.connect();
        RedisCommands<String, String> redisCommands = connection.sync();

        String lock = redisCommands.get(lockKey);
        if (lock != null) {
            return Boolean.FALSE;
        } else {
             返回 null, key 已经存在了, 说明有其他客户端持有锁
            String setValue = redisCommands.set(lockKey, lockValue, SetArgs.Builder.nx().ex(lockTimeout));
            if (setValue == null) {
                return Boolean.FALSE;
            } else {
                return Boolean.TRUE;
            }
        }
    } catch (Exception e) {
        log.error("获取锁错误, error = {}", e);
        return Boolean.FALSE;
    }
}
```

# 解锁, 使用 Lua 脚本保证操作原子性

```java
public boolean unlock(String lockKey, String lockValue) {
    try {
        StatefulRedisConnection<String, String> connection = redisClient.connect();
        RedisCommands<String, String> redisCommands = connection.sync();
        // String scriptTemplate = "if redis.call('get', '%s') == '%s' then return redis.call('del', '%s') else return 0 end";
        // String script = String.format(scriptTemplate, lockKey, lockValue, lockKey);
        // long result = redisCommands.eval(script, ScriptOutputType.INTEGER);
        String scriptTemplate = "if redis.call('get', KEYS[1]) == ARGV[1] then return redis.call('del', KEYS[1]) else return 0 end";
        long result = redisCommands.eval(scriptTemplate, ScriptOutputType.INTEGER, new String[]{lockKey}, lockValue);
        return result == 1;
    } catch (Exception e) {
        log.error("释放锁错误, error = {}", e);
        return Boolean.FALSE;
    }
}
```

# RedLock 分布式锁

上面的情况都是在 Redis 单机的时候的分布式锁, 尤其是第三种情况是一种较为安全的分布式锁方案, 在集群下模式下 Redis 作者提出了 RedLock 的解决方,RedLock 的实现思路:

* 取得当前时间
* 使用上文提到的第三种方案依次获取N个节点的 Redis 锁
* 如果获取到的锁的数量大于 (N / 2 + 1) 个, 且获取的时间小于锁的有效时间就认为获取到了一个有效的锁, 锁自动释放时间就是最初的锁释放时间减去之前获取锁所消耗的时间
* 如果获取锁的数量小于 (N / 2 + 1), 或者在锁的有效时间内没有获取到足够的说, 就认为获取锁失败, 这个时候需要向所有节点发送释放锁的消息

关于 RedLock 分布式锁的实现, 官方推荐 [Redisson](https://github.com/redisson/redisson) 客户端

# 参考资料

* [Redis分布式锁解决方案](https://juejin.im/post/5b8737cdf265da43737ea13a)
* [Redis实现分布式锁](https:www.xilidou.com/2017/10/23/Redis%E5%AE%9E%E7%8E%B0%E5%88%86%E5%B8%83%E5%BC%8F%E9%94%81/) - 关于 RedLock 的实现思路
* [基于Redis的分布式锁到底安全吗 (上)](http:zhangtielei.com/posts/blog-redlock-reasoning.html) - Martin 和 Antirez 对 RedLock 的讨论
* [基于Redis的分布式锁到底安全吗 (下)](http:zhangtielei.com/posts/blog-redlock-reasoning-part2.html) - Martin 和 Antirez 对 RedLock 的讨论