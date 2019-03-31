---
title: Redis-Redis有序集合
date: 2019-02-16 21:26:51
categories: Redis
---

Redis 有序集合和集合一样不会出现重复的元素, 和集合不同的是, 有序集合需要额外设置一个 score 值, 这个 score 值被用作于排序

# ZADD 添加元素

* ZADD key score member [score member ...]
* 有序集合添加元素, 在 2.4 之前的版本只允许添加单个元素

```java
redisCommands.zadd("zset", 3.0, "C", 1.0, "A", 2.0, "B");
log.info("zset = {}", redisCommands.zrange("zset", 0, -1).toString());
redisCommands.del("zset");
```

# ZRANGE, ZREVRANGE 获取集合

* ZRANGE key start stop [WITHSCORES] - 递增获取
* ZREVRANGE key start stop [WITHSCORES] - 递减获取
* 时间复杂度 O(log(N)+M), N 为有序集合包含的元素数量, M 为结果集的元素数量

```java
long addSize = redisCommands.zadd("zset", ScoredValue.from(1, Optional.of("A")),
        ScoredValue.from(2, Optional.of("B")),
        ScoredValue.from(3, Optional.of("C")),
        ScoredValue.from(4, Optional.of("C")),
        ScoredValue.from(5, Optional.of("D")));
```

<!-- more -->

# ZRANGE key start stop, 获取集合

```java
// 重复添加的元素只算成功添加一个
log.info("addSize = {}", addSize);
log.info("zset = {}", redisCommands.zrange("zset", 0, -1).toString());
```

# ZRANGE key start stop WITHSCORES, 获取集合, 并返回 scored

```java
List<ScoredValue<String>> sourceValues = redisCommands.zrangeWithScores("zset", 0, -1);
log.info("zset = {}", sourceValues.toString());
redisCommands.del("zset");
```

# ZRANGEBYSCORE, ZRANGEBYSCORE 获取集合

* ZRANGEBYSCORE key min max [WITHSCORES] [LIMIT offset count] - 递增获取
* ZREVRANGEBYSCORE key max min [WITHSCORES] [LIMIT offset count] - 递减获取
* 获取 score 在 [min max] 区间间的集合
* 可以添加 LIMIT 参数, 对返回结果进行分页, 效果和 MYSQL 中的语句一样
* 时间复杂度 O(log(N)+M), N 为有序集合包含的元素数量, M 为被结果集的基数

```java
redisCommands.zadd("zset", ScoredValue.from(1, Optional.of("A")),
        ScoredValue.from(2, Optional.of("B")),
        ScoredValue.from(3, Optional.of("C")),
        ScoredValue.from(4, Optional.of("D")),
        ScoredValue.from(5, Optional.of("E")),
        ScoredValue.from(6, Optional.of("F")));
List<String> dataList1 = redisCommands.zrangebyscore("zset", Range.create(2, 5));
log.info("zset = {}", dataList1.toString());

// 和 dataList1 相比, dataList2 只获取到 D, E 两个元素
List<String> dataList2 = redisCommands.zrangebyscore("zset", Range.create(2, 5), Limit.create(2, 2));
log.info("zset = {}", dataList2.toString());
redisCommands.del("zset");
```

# ZRANGEBYLEX, ZREVRANGEBYLEX 获取集合

* ZRANGEBYLEX key min max [LIMIT offset count] - 递增获取
* ZREVRANGEBYLEX key min max [LIMIT offset count] - 递减获取
* 参考资料: [http://www.redis.cn/commands/zrangebylex.html](http://www.redis.cn/commands/zrangebylex.html)
* 对集合内 score 相同的元素进行字典排序, 如果集合中有 score 不同, 返回的排序结果会不准确, 使用 min 和 max 指定返回结合的范围
* [min, [max, 表示元素值区间在 [min, max] 的集合
* min, (max, 表示元素值区间在 [min, max) 的集合
* (min, [max, 表示元素值区间在 (min, max] 的集合
* (min, (max, 表示元素值区间在 (min, max) 的集合
* -, +, 表示获取全部
* 时间复杂度 O(log(N)+M), N 为有序集合包含的元素数量, M 为被结果集的基数
* 该指令常用于按字符串排序场景

```java
// 下例获取元素值在 [B, F] 区间的集合
redisCommands.zadd("zset", ScoredValue.from(1, Optional.of("A")),
        ScoredValue.from(1, Optional.of("B")),
        ScoredValue.from(1, Optional.of("C")),
        ScoredValue.from(1, Optional.of("D")),
        ScoredValue.from(1, Optional.of("E")),
        ScoredValue.from(1, Optional.of("F")));
log.info("zset = {}", redisCommands.zrangebylex("zset", Range.create("B", "F")));
redisCommands.del("zset");

// 下例获取元素值在 [B, F] 区间的集合, 但是由于 score 不同, 导致获取错误的集合
redisCommands.zadd("zset", ScoredValue.from(2, Optional.of("A")),
        ScoredValue.from(2, Optional.of("B")),
        ScoredValue.from(1, Optional.of("C")),
        ScoredValue.from(1, Optional.of("D")),
        ScoredValue.from(1, Optional.of("E")),
        ScoredValue.from(1, Optional.of("F")));
log.info("zset = {}", redisCommands.zrangebylex("zset", Range.create("B", "F")));
redisCommands.del("zset");
```

# ZRANK, ZREVRANK 获取排名

* ZRANK key member - 顺序
* ZREVRANK key member - 逆序
* 获取元素在集合中的排名, 排名从 0 开始为第 1 名
* 时间复杂度 O(log(N)), N 为有序集合包含的元素数量

```java
redisCommands.zadd("zset", ScoredValue.from(1, Optional.of("A")),
        ScoredValue.from(2, Optional.of("B")),
        ScoredValue.from(3, Optional.of("C")),
        ScoredValue.from(4, Optional.of("D")),
        ScoredValue.from(5, Optional.of("E")));
log.info("rank = {}", redisCommands.zrank("zset", "B"));
log.info("rank = {}", redisCommands.zrevrank("zset", "B"));
```

# 删除元素

* ZREM key member [member …]
* 删除指定的元素, 返回成功移除的数量
* 时间复杂度 O(M*log(N)), N 为有序集合包含的元素数量, M 为被成功移除的成员的数量

```java
redisCommands.zrem("zset", "D", "E");
```

* ZREMRANGEBYRANK key start stop
* 删除指定下标在区间 [start, stop] 区间内的元素, 返回成功移除的数量
* 时间复杂度 O(M*log(N)), N 为有序集合包含的元素数量, M 为被成功移除的成员的数量

```java
redisCommands.zremrangebyrank("zset", 0, 1);
```

* ZREMRANGEBYSCORE key min max
* 删除 score 在 [min, max] 区间内的元素, 返回成功移除的数量
* 时间复杂度 O(M*log(N)), N 为有序集合包含的元素数量, M 为被成功移除的成员的数量

```java
redisCommands.zremrangebyscore("zset", Range.create(1, 4));
```

* ZREMRANGEBYLEX key min max
* 删除一个 score 相同的集合中元素值在 min 和 max 之间的元素, min 和 max 的含义和 ZRANGEBYLEX 一样
* 时间复杂度 O(M*log(N)), N 为有序集合包含的元素数量, M 为被成功移除的成员的数量

```java
redisCommands.zremrangebylex("zset", Range.create("B", "D"));
```

# 获取指定元素分数

* ZSCORE key member
* 返回有序集合中指定元素的 score
* 时间复杂度 O(log(N)), N 为有序集合包含的元素数量

```java
redisCommands.zscore("zset", "A");
```

# ZINCRBY 增加指定元素分数

* ZINCRBY key increment member
* 给指定元素的分数增减或者减少 increment 值, 可以是整数或者浮点数, 返回元素新的 score 值
* 时间复杂度 O(log(N)), N 为有序集合包含的元素数量

```java
redisCommands.zincrby("zset", 1.0, "A");
```

# 获取集合大小

* ZCARD key
* 获取集合的大小
* 时间复杂度 O(1)

```java
redisCommands.zcard("zset");
```

* ZCOUNT key min max
* 获取指定 score 区间内的集合的大小
* 时间复杂度 O(log(N)), N 为有序集合包含的元素数量

```java
redisCommands.zcount("zset", Range.create(2, 4));
```

* ZLEXCOUNT key min max
* 获取在一个 score 相同的集合中元素值在 min 和 max 之间的元素集合的大小, min 和 max 的含义和 ZRANGEBYLEX 一样
* 时间复杂度 O(log(N)), N 为有序集合包含的元素数量

```java
redisCommands.zlexcount("zset", Range.create("B", "F"));
```

# 求交集和并集

有序集合交集和并集和集合的类似, 就不贴测试代码了

* 参考资料: [http://www.redis.cn/commands/zinterstore.html](http://www.redis.cn/commands/zinterstore.html)
* 参考资料: [http://www.redis.cn/commands/zunionstore.html](http://www.redis.cn/commands/zunionstore.html)