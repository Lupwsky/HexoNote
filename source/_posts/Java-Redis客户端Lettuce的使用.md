---
title: Java-Redis客户端Lettuce的使用
date: 2019-01-26 23:03:57
categories: Java
---

# Redis 客户端 Lettuce 的使用

Lettuce 是一个 Java 实现的 Redis 客户端, Lettuce 相对于 Jedis, 除了和 Jedis 一样提供了对 Redis 基础的操作外, 在多线程下也是实例安全, 并且提供了同步, 异步和响应式的调用, 同时许多接口都支持 Stream API, 对大的大的数据集合的处理可以减少内存的消耗,非常方便等的, Lettuce 具备了更加优秀的特性

## 项目中引入依赖

```xml
<dependency>
    <groupId>io.lettuce</groupId>
    <artifactId>lettuce-core</artifactId>
    <version>5.1.3.RELEASE</version>
</dependency>
```

<!-- more -->

## 建立链接

使用 Lettuce 客户端连接 Redis 创建 RedisClient 实例对象, 主要需要使用 RedisURL 这个类, 支持单节点 (Redis Standalone), 单节点 SSL ( Redis Standalone SSL), 独立的 UNIX 域套接字 (Redis Standalone Unix Domain Socket) 和哨兵模式连接 (Redis Sentinel), 一个单节点的连接模式的使用示例如下:

```java
// RedisURL 也提供了建造者模式来创建实例
// RedisURI.create("10.98.164.156", 6380).withPassword("lupengwei.4585").build();
RedisURI redisURI = RedisURI.create("10.98.164.156", 6380);
redisURI.setPassword("lupengwei.4585");
redisURI.setTimeout(Duration.ofSeconds(10));
redisURI.setDatabase(0);
RedisClient redisClient = RedisClient.create(redisURI);

StatefulRedisConnection<String, String> connection = redisClient.connect();
RedisCommands<String, String> syncCommands = connection.sync();
syncCommands.set("username", "lpw");
String value = syncCommands.get("username");
connection.close();
redisClient.shutdown();
```

除了使用 RedisURL 类外, 还可以使用符合 URL 格式的字符串, 然后调用 RedisClient.create(String url) 连接 Redis 创建 RedisClient 实例对象, 各模式连接 URL 格式分别如下:

```text
redis://[:password@] host [:port] [/database] [?[timeout=timeout[d|h|m|s|ms|us|ns]] [&_database=database_]]
rediss://[:password@] host [:port] [/database] [?[timeout=timeout[d|h|m|s|ms|us|ns]] [&_database=database_]]
redis-socket:// path [?[timeout=timeout[d|h|m|s|ms|us|ns]] [&_database=database_]]
redis-sentinel://[: password@] host1 [:port1] [,host2[:port2]] [,hostN[:portN]] [/database] [?[timeout=timeout[d|h|m|s|ms|us|ns]] [&_sentinelMasterId=sentinelMasterId_] [&_database=database_]]
```

其中的 `[d|h|m|s|ms|us|ns]` 部分时超时时间单位可选值, 分别对应 `[天|时|分|秒|毫秒|微妙|纳秒]`, 下面是一个简单的使用示例:

```java
RedisClient redisClient = RedisClient.create("redis://lupengwei.4585@10.98.164.156:6380/0?timeout=10s");
StatefulRedisConnection<String, String> connection = redisClient.connect();
RedisCommands<String, String> syncCommands = connection.sync();
syncCommands.set("username", "lpw");
```

更加详细信息可参考官方文档: `https://github.com/lettuce-io/lettuce-core/wiki/Redis-URI-and-connection-details`

## 同步调用

同步的调用很简单, 在上面的例子中已经有过调用, 在获取到 连接后, 调用 connection.sync() 方法就可以获取到 RedisCommands 对象实例, 然后就可以像 Redis 一样进行各种基础操作了, 使用示例如下, 可以看到其调用方法是和 Redis 指令操作是类似的:

```java
RedisCommands<String, String> syncCommands = connection.sync();
syncCommands.set("username", "lpw");
// 设置超时时间
syncCommands.expire("username", 3000);
// 获取超时时间
long timeout = syncCommands.ttl("username");
String value = syncCommands.get("username");
```

一般在 Spring 项目中配置成对应的 Bean, 代码如下

```java
@Slf4j
@Configuration
public class RedisConfiguration {

    @Bean
    public RedisClient getRedisClient() {
        return RedisClient.create("redis://lupengwei.4585@10.98.164.156:6380/0?timeout=10s");
    }

    @Bean
    RedisCommands<String, String> getStringSyncRedisCommands(RedisClient redisClient) {
        return redisClient.connect().sync();
    }
}

@Slf4j
@Service
public class RedisServiceImpl {

    private final RedisCommands<String, String> stringSyncRedisCommands;

    @Autowired
    public RedisServiceImpl(RedisCommands<String, String> stringSyncRedisCommands) {
        this.stringSyncRedisCommands = stringSyncRedisCommands;
    }

    void lettuceRedisTest() {
        stringSyncRedisCommands.set("username", "lpw");
        stringSyncRedisCommands.expire("username", 3000);
        long timeout = stringSyncRedisCommands.ttl("username");
        String value = stringSyncRedisCommands.get("username");
        log.info("[Lettuce Redis] timeout = {}, value = {}", timeout, value);
    }
}
```

## 异步调用

在使用 Jedis 的时候, 需要异步调用时需要借助 Future 或者 JDK 1.8 中的 CompleteableFuture 类来实现, Lettuce 对此进行封装, 对于异步调用非常方便

```java
RedisAsyncCommands<String, String> asyncCommands = connection.async();
RedisFuture<String> future = asyncCommands.get("username");

// TODO 其他耗时任务

try {
    String username = future.get();
} catch (InterruptedException | ExecutionException e) {
    e.printStackTrace();
}

// 等待多个 RedisFuture 执行完成
List<RedisFuture<String>> futures = new ArrayList<RedisFuture<String>>();
for (int i = 0; i < 10; i++) {
    futures.add(asyncCommands.get("username" + i, ""));
}
LettuceFutures.awaitAll(1, TimeUnit.MINUTES, futures.toArray(new RedisFuture[futures.size()]));
```

RedisFuture 的使用和 JDK 1.8 中的 CompleteableFuture 使用类似, 支持 JDK 1.8 中各种新特性, 更加详细的使用方法可以参考官方资料: `https://github.com/lettuce-io/lettuce-core/wiki/Asynchronous-API`

## 发布订阅功能 (观察者模式)

熟悉 Redis 的应该知道 Redis 支持发布订阅功能, 使用 SUBSCRIBE 和 PUBLISH 指令就可以实现, Lettuce 也提供了发布订阅功能相关 API, 使用起来也很方便, Jedis 也有提供相关 API, 参考资料 [Jedis 实现 Publish/Subscribe 功能](https://blog.csdn.net/lihao21/article/details/48370687), [Redis 用作消息队列和发布订阅模式](https://zhuanlan.zhihu.com/p/52734627)

```java

```

# 参考资料

* [Redis4.0 到来, Jedis不能用了咋整](https://zhuanlan.zhihu.com/p/32348602)
* [Lettuce 和Jedis的基准测试](https://juejin.im/post/5bc6d5d2e51d450e7a2541e8)
* [高级的 Redis Java客户端 - Lettuce](https://cloud.tencent.com/developer/article/1142355)