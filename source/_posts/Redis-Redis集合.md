---
title: Redis-Redis集合
date: 2019-02-16 21:18:44
categories: Redis
---

# SADD 添加元素

* SADD key member [member …]
* 添加元素到集合中, 返回被添加到集合中的新元素的数量

```java
// 下例集合的值为 [1, 2], 重复的元素不会被添加
redisCommands.sadd("set", "1", "1", "2");
log.info("set = {}", redisCommands.smembers("set").toString());
redisCommands.del("set");
```

# SISMEMBER 判断元素是否在集合中

* SISMEMBER key member
* 判断 member 是否时集合的元素, 0 = 否, 1 = 是, 当集合不存在是返回的也是 0

```java
// 下例将会输出 true
redisCommands.sadd("set", "1", "2");
log.info("value = {}", redisCommands.sismember("set", "1"));
redisCommands.del("set");
```

<!-- more -->

# SPOP 随机弹出一个元素

* SPOP key
* 移除并返回集合中的一个随机元素

```java
redisCommands.sadd("set", "1", "2");
redisCommands.spop("set");
log.info("set = {}", redisCommands.smembers("set").toString());
redisCommands.del("set");
```

# SRANDMEMBER 随机获取元素

* SRANDMEMBER key [count]
* 随机返回元素, 在 2.6 之后可以添加参数 count, count 表示返回的元素个数
* 时间复杂度 O(N), N 值为 count
* 当不提供 count 参数时, 若集合不存在, 返回 null, 提供了 count 时, 若集合为空, 则返回一个空集合

```java
        redisCommands.sadd("set", "1", "2", "3", "4", "5");
        List<String> setList = redisCommands.srandmember("set", 3);
        String setValue = redisCommands.srandmember("set");
        log.info("set = {}", setList.toString());
        log.info("value = {}", setValue);
        redisCommands.del("set");
```

# SREM 移除指定元素

* SREM key member [member …]
* 移除元素, 并返回成功移除元素的数量
* 时间复杂度为 O(N), N 为要移除元素的数量

```java
redisCommands.sadd("set", "1", "2", "3", "4", "5");
redisCommands.srem("set", "2", "3");
log.info("set = {}", redisCommands.smembers("set").toString());
redisCommands.del("set");
```

# SMOVE 移动元素到其他集合中

* SMOVE set1 set2 member
* 将集合 1 的元素移动到集合 2 中, 并删除集合 1 中的元素, 成功返回 1

```java
// 下例 set2 输出的值为中包含 6, 5, 7 三个元素
redisCommands.sadd("set1", "1", "2", "3", "4", "5");
redisCommands.sadd("set2", "6", "7");
redisCommands.smove("set1", "set2", "5");
log.info("set = {}", redisCommands.smembers("set").toString());
redisCommands.del("set1", "set2");
```

# SCARD 获取集合大小

* SCARD key
* 返回集合大小, 集合为空时返回 0

```java
redisCommands.sadd("set", "1", "2", "3", "4", "5");
log.info("size = {}", redisCommands.scard("set"));
redisCommands.del("set");
```

# SMEMBERS 获取集合的全部元素

* SMEMBERS key
* 返回所有集合的元素
* 时间复杂度 O(N), N 为集合元素的数量, 生产环境上请慎用

```java
redisCommands.sadd("set", "1", "2", "3");
log.info("value = {}", redisCommands.sismember("set", "1"));
redisCommands.del("set");
```

# SSCAN 获取集合中符合条件的元素

* SCAN cursor [MATCH pattern] [COUNT count]
* 参考资料: [http://www.redis.cn/commands/scan.html](http://www.redis.cn/commands/scan.html)
* 参考资料: [https://juejin.im/post/5bbcc8325188255c74553ae3](https://juejin.im/post/5bbcc8325188255c74553ae3)
* 该指令 2.8 提供, 返回集合中符合条件的元素
* 时间复杂度 O(N), N 为匹配到的元素的数量

```java
redisCommands.sadd("set", "A1", "A2", "B1", "B2", "A3", "C1", "D1");

// 全匹配, 生产环境慎用, 这个例子包含 [A1, A2, B1, D1, C1, B2, A3] 6 个元素
ValueScanCursor<String> valueScanCursor1 = redisCommands.sscan("set");
List<String> scanList1 = valueScanCursor1.getValues();
log.info("set = {}", scanList1.toString());

// 匹配元素值开头为 A 的元素, 这个例子包含 [A1, A2, A3] 三个元素
ValueScanCursor<String> valueScanCursor2 = redisCommands.sscan("set",
        ScanArgs.Builder.matches("A*"));
List<String> scanList2 = valueScanCursor2.getValues();
log.info("set = {}", scanList2.toString());

// count 元素, 随机的返回集合中的指定 count 个元素, 下例返回两个元素
ValueScanCursor<String> valueScanCursor3 = redisCommands.sscan("set",
        ScanArgs.Builder.matches("A*").limit(2));
List<String> scanList3 = valueScanCursor3.getValues();
log.info("set = {}", scanList3.toString());
redisCommands.del("set");
```

# SINTER 多个集合交集

* SINTER key [key …]
* 求多个集合交集
* 时间复杂度 O(N * M), N 为求交集的集合中最小的那个 SIZE, M 为集合的个数

```java
// 下例返回 A1
redisCommands.sadd("set1", "A1", "A2");
redisCommands.sadd("set2", "A1", "B1");
redisCommands.sadd("set3", "A1", "C1");
log.info("set = {}", redisCommands.sinter("set1", "set2", "set3").toString());
redisCommands.del("set1", "set2", "set3");
```

# SINTERSTORE 多个集合交集

* SINTERSTORE destination key [key …]
* 求多个集合的交集
* 时间复杂度 O(N * M), N 为求交集的集合中最小的那个 SIZE, M 为集合的个数
* 和 SINTER 不同, SINTER 将交集返回给客户端, SINTERSTORE将交集保存到 destination 集合中, 如果 destination 存在则会覆盖
* 返回值为集合中元素的数量

```java
redisCommands.sadd("set1", "A1", "A2");
redisCommands.sadd("set2", "A1", "B1");
redisCommands.sadd("set3", "A1", "C1");
log.info("size = {}", redisCommands.sinterstore("resultSet", "set1", "set2", "set3"));
log.info("set = {}", redisCommands.smembers("resultSet").toString());
redisCommands.del("set1", "set2", "set3", "resultSet");
```

# SUNION, SUNIONSTORE 求并集

* 用法和 SINTER, SINTERSTORE 类似, 不再举例
* 时间复杂度 O(N), N 为所有集合元素数量之和

# SDIFF, SDIFFSTORE 求差集

* 用法和 SINTER, SINTERSTORE 类似, 不再举例
* 时间复杂度 O(N), N 为所有集合元素数量之和