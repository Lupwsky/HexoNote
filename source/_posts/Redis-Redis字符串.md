---
title: Redis-Redis字符串
date: 2019-02-16 20:31:15
categories: Redis
---

注: 使用 Redis 的客户端 Lettuce 进行的测试, 使用方法见 [Redis-Redis客户端Lettuce的使用]()

```java
StatefulRedisConnection<String, String> connection = redisClient.connect();
RedisCommands<String, String> redisCommands = connection.sync();
```

# SET

`set key value`

* 对已有值的 key 会覆写旧值, 无视类型
* 对带有 ttl 的 key, 此操作会清除和替代原有 ttl

```java
// redisCommands 的大多数 set 方法成功返回字符串 OK, 失败返回 null
redisCommands.set("username-set", "lpw-set");
```

<!-- more -->

# MSET

`mset key1 value1, key2, key2`

* msetnx 批量添加键值, 注意批量设置 Redis 没有提供设置过期时间的接口, 如果需要设置过期时间使用 set 循环添加

```java
Map<String, String> map = Maps.newHashMap();
map.put("username-mset", "lpw-mset");
map.put("password-mset", "lpw-pwd-mset");
redisCommands.mset(map);
redisCommands.msetnx(map);
```

# SET 的参数

Redis 2.6.12 版本开始, set 命令可以在后面添加一些参数, 如下:

* ex second, 设置键的过期时间单位为秒, second 为时间参数, 例 set key value ex 5, 等同于 setex key 5 value (set key 5 value 会报错)
* px millisecond, 设置键的过期时间单位为毫秒, millisecond 为时间参数, 例 set key value px 5000, 等同于 psetex key 5000 value
* nx, key 不存在时进行设置操作, 存在时则不会进行任何操作, 等同于 setnx key value
* xx, key 已经存在时才对键进行设置操作, 注意 Redis 没有 setxx 指令

```java
redisCommands.set("username-set-args", "lpw-set-args", SetArgs.Builder.px(5000));
```

# SETEX

`setex key second value`

setex 设置过期时间, 单位为秒:

```java
redisCommands.setex("username-setex", 10,"lpw-setex");
```

set 设置过期时间, set 值后再使用 expire 设置过期时间, 不同的是 setex 是原子操作, set 值后再使用 expire 设置过期时间是非原子操作

```java
redisCommands.set("username-expire", "lpw-expire");
redisCommands.expire("username-expire", 10);
```

# PSETEX

`setex key millisecond value`

对键进行设置操作并设置过期时间, 时间单位为毫秒

```java
redisCommands.psetex("username-psetex", 10000, "lpw-psetex");
```

# SETNX KEY

不存在时进行设置操作, key 存在时则不会进行任何操作

```java
redisCommands.set("username-setnx", "lpw-setnx");
```

# GETSET

`getset key value`

* 将给定 key 的值设为 value, 并返回 key 的旧值
* 如果不是字符串类型, 则报错, 如果 key 值不存在返回 null

```java
redisCommands.getset("username-set", "lpw-get-set");
```

# APPEND

* 将 value 追加到 key 原来的值的末尾
* 不存在就新建, 返回最终 value 的长度
* 不是字符串类型则报错, redisCommands.append 方法会抛出异常还是返回 0 ? 等待验证

```java
long appendResult = redisCommands.append("username-set", "-append");
log.info("appendResult = {}", appendResult);
```

# INCR

* 如果值是一个整数类型, 该指令可以让值加 1, 但是最大的值不不能超过 long 类型的大小, incr key
* 如果 key 值不存在, 则新建 key 并设置值为 0 后再加 1, 该指令返回最终的值

```java
redisCommands.set("count", "20");
long incrResult = redisCommands.incr("count");
```

运用这种特性可以完成类似计数器, 限流功能, [http://redisdoc.com/string/incr.html](http://redisdoc.com/string/incr.html)

# INCRBY

* 如果值是一个整数类型, 该指令可以让值加指定的数值, incrby key num
* 如果 key 值不存在, 则新建 key 并设置值为 0 后再加 1, 该指令返回最终的值

```java
redisCommands.set("count", "20");
long incrByResult = redisCommands.incrby("count", 10);
```

# INCRFLOAT

* 将值和浮点型的数相加, incrfloat key floatnNum
* 使用 redis 时返回的时字符串, redisCommands.incrbyfloat 方法进行了封装, 返回的是浮点型数据
* 关于 incrfloat, [http://redisdoc.com/string/incrbyfloat.html](http://redisdoc.com/string/incrbyfloat.html)

```java
redisCommands.set("count", "20");
Double incrbyfloatResult = redisCommands.incrbyfloat("count", 5.5);
```

# DECR, DECRBY 和 DECRBYFLOAT

和 incr, incrby 相反, 减去数值, `但是要注意的是没有 decrbyfloat 指令`