---
title: Redis-Redis位图
date: 2019-02-16 20:54:44
categories: Redis
---

关于位图: [https://juejin.im/book/5afc2e5f6fb9a07a9b362527/section/5b330620e51d4558e03ce7f8](https://juejin.im/book/5afc2e5f6fb9a07a9b362527/section/5b330620e51d4558e03ce7f8)

```java
StatefulRedisConnection<String, String> connection = redisClient.connect();
RedisCommands<String, String> redisCommands = connection.sync();
```

# 设置值-零存

设置的时候从高位往低位设置

```java
redisCommands.setbit("bitmap", 0, 0);
redisCommands.setbit("bitmap", 1, 1);
redisCommands.setbit("bitmap", 2, 1);
redisCommands.setbit("bitmap", 4, 1);
redisCommands.setbit("bitmap", 9, 1);
redisCommands.setbit("bitmap", 10, 1);
redisCommands.setbit("bitmap", 13, 1);
redisCommands.setbit("bitmap", 15, 1);
```

<!-- more -->

# 设置值-整存

整存的方法和存放普通的字符串并无差别, 如下, 和上面使用零存的方式存入的值时一致的

```java
redisCommands.set("bitmap", "he");
```

# 获取值-整取

```java
og.info("value = {}", redisCommands.get("bitmap"));

// 读取的时候从低位的第一位非 0 位开始的往高位读取, 高位不足的部分补零
// 将上例中的第 16 位设置为 0, 结果是一样的
redisCommands.setbit("bitmap", 16, 0);
log.info("value = {}", redisCommands.get("bitmap"));

// offset 参数必须大于或等于 0 且小于 2^32, bit 映射被限制在 512 MB 之内
// offset 越大消耗的内存也越大, 尽量避免大的 offset 的值, 大量的大内存分配很可能造成 Redis 服务器被阻塞
```

# 获取值-零取

获取指定 key 所存储的字符串的偏移量的位的值, 0 或者 1, 当 key 不存在或者 offset 超出当前的位数范围, 均返回 0

```java
log.info("value = {}", redisCommands.getbit("bitmap", 1));
```

# BITCOUNT

`BITCOUNT key [start] [end]`, 时间复杂度 O(N)

* [http://redisdoc.com/bitmap/bitcount.html](http://redisdoc.com/bitmap/bitcount.html)
* bitcount 指令用于统计给定的 key 对应的值的比特位被设置为 1 的数量

```java
// 如上例中, 字符串 "he" 的比特位被设置为 1 的数量为 7
log.info("value = {}", redisCommands.bitcount("bitmap"));
```

可以设置参数 [start, end], 统计指定字节范围内比特位被设置为 1 的数量:

```java
// 字符串 "he" 的第 0 字节开始位至第 0 字节结束位的比特位被设置为 1 的数量为 3
// 字符串 "he" 的第 1 字节开始位至第 1 字节结束位的比特位被设置为 1 的数量为 4
// 字符串 "he" 的第 0 字节开始位至第 1 字节结束位的比特位被设置为 1 的数量为 7
// 字符串 "he" 的第 1 字节开始位至第 2 字节结束位的比特位被设置为 1 的数量为 4, 这种情况和第二种一样, 统计时对超出的字节补 0 计算
log.info("value = {}", redisCommands.bitcount("bitmap", 0, 0));
log.info("value = {}", redisCommands.bitcount("bitmap", 1, 1));
log.info("value = {}", redisCommands.bitcount("bitmap", 0, 1));
log.info("value = {}", redisCommands.bitcount("bitmap", 1, 2));
```

参数 [start, end] 可以为负值, 如 -1 表示倒数第一个字节:

```java
log.info("value = {}", redisCommands.bitcount("bitmap", 0, -1));
```

# BITOPS

`BITOPS key bit [start] [end]`, 时间复杂度 O(N)

* 参考资料: [http://redisdoc.com/bitmap/bitpos.html](http://redisdoc.com/bitmap/bitpos.html)
* 返回第一个比特位为 1 或者为 0 的位置, 如果 key 对应的值不存在返回 -1
* 也可以设置参数 [start, end], 返回第一个比特位为 1 或者为 0 的位置

```java
redisCommands.setbit("bitops", 3, 1);
redisCommands.setbit("bitops", 10, 1);
redisCommands.setbit("bitops", 15, 1);

// 返回第一个比特位为 1 的位置, 此处值为 3
log.info("value = {}", redisCommands.bitpos("bitops", true));

// 返回第一个比特位为 0 的位置, 此处值为 0
log.info("value = {}", redisCommands.bitpos("bitops", true));

// key 对应的值不存在返回 -1
log.info("value = {}", redisCommands.bitpos("none", true));
```

# BITOP

参考资料: [http://redisdoc.com/bitmap/bitop.html](http://redisdoc.com/bitmap/bitop.html)

* BITOP AND destkey key [key ...], 对一个或多个 key 求逻辑并，并将结果保存到 destkey, 时间复杂度 O(N)
* BITOP OR destkey key [key ...], 对一个或多个 key 求逻辑或，并将结果保存到 destkey, 时间复杂度 O(N)
* BITOP XOR destkey key [key ...], 对一个或多个 key 求逻辑异或，并将结果保存到 destkey, 时间复杂度 O(N)
* BITOP NOT destkey key, 对给定 key 求逻辑非，并将结果保存到 destkey, 时间复杂度 O(N)
* 除了 NOT 操作之外, 其他操作都可以接受一个或多个 key 作为输入
* 当 BITOP 处理不同长度的字符串时, 较短的那个字符串或者对应的 key 没有值字符串, 所缺少的部分会被看作 0
* 以上指令最终保存的字符串长度是输入的 key 对应的最长的字符串的长度

```java
redisCommands.bitopOr("destkey", "bitmap", "bitops");
```