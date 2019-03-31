---
title: Redis-Redis列表
date: 2019-02-16 21:07:48
categories: Redis
---

# LPUSH

`LPUSH key value [value …]`

左侧压入列表 (每次从队列头压入数据), 返回当前列表的长度, 如果列表不存在就会创建, 列表允许重复的元素:

```java
// list 的值为 [d, d, c, b, a]
redisCommands.lpush("list", "a", "b", "c", "d", "d");
log.info("list = {}", redisCommands.lrange("list", 0, -1));
redisCommands.del("list");
```

<!-- more -->

# RPUSH

和 LPUSH 相反, 右侧压入列表 (每次从队列尾部压入数据)

# LPUSHX

左侧压入列表 (每次从队列头压入数据), 返回当前列表的长度, 和 LPUSH 不同, 当列表不存在是不会创建列表和压入数据, 且返回 0

```java
// 下例将输出 [a+, b, a]
redisCommands.lpush("list", "a", "b");
redisCommands.lpushx("list", "a+");
log.info("list = {}", redisCommands.lrange("list", 0, -1));
redisCommands.del("list");
```

# RPUSHX

和 LPUSHX 相反, 右侧压入列表 (每次从队列尾部压入数据), 就不贴测试代码了

# LRANGE

`LRANGE key start stop`

* 时间复杂度 O(S + N), S 为偏移量, N 为元素个数, 获取长列表使用时要注意
* 如果 start 值超出列表值, 返回空数组
* 如果 stop 值超出列表值, stop 值会被设置为列表的长度值

```java
redisCommands.lpush("list", "a", "b", "c", "d", "d");
List<String> list = redisCommands.lrange("list", 0, -1);
log.info("list = {}", list.toString());
redisCommands.del("list");
```

# LLEN

获取列表长度, 列表不存在返回 0

```java
redisCommands.rpush("list", "a", "b", "c", "d", "d");
log.info("size = {}", redisCommands.llen("list"));
```

# LPOP

* 从队列头部弹出数据, 类似栈, 获取到的元素会在队列中被删除掉
* 如果队列为空 (队列不存在), 返回 null

```java
redisCommands.lpush("list", "a", "b");
// 输出 b
log.info("value = {}", redisCommands.lpop("list"));
// 输出 a
log.info("value = {}", redisCommands.lpop("list"));
// 输出 null
log.info("value = {}", redisCommands.lpop("list"));
redisCommands.del("list");
```

# RPOP

和 LPOP 用法一样, 但是从队列尾部弹出数据

# RPOPLPUSH

`RPOPLPUSH source destination`

* 参考资料: [http://redisdoc.com/list/rpoplpush.html](http://redisdoc.com/list/rpoplpush.html)
* 将列表 source 中的最后一个元素弹出, 并返回, 同时将弹出的元素放到队列 destination 的头部
* 如果 source 和 destination 是同一个队列, 则可实现列表的旋转操作
* 如果第一个列表为空, 则返回 null, 也不会将有任何元素放到队列 destination 的头部

```java
// 下例 list1 的值将会是 [a1], list2 的值将会是 [b1, a2, b2]
redisCommands.rpush("list1", "a1", "b1");
redisCommands.rpush("list2", "a2", "b2");
redisCommands.rpoplpush("list1", "list2");
log.info("list1 = {}", redisCommands.lrange("list1", 0, -1));
log.info("list2 = {}", redisCommands.lrange("list2", 0, -1));
redisCommands.del("list1");
redisCommands.del("list2");
```

# BLPOP, BRPOP 和 BRPOPLPUSH

`BLPOP key [key …] timeout`

* 参考资料: [http://redisdoc.com/list/blpop.html](http://redisdoc.com/list/blpop.html)
* 它是 LPOP 命令的阻塞版本, 当给定列表内没有任何元素可供弹出的时候
* 连接将被 BLPOP 命令阻塞, 直到等待超时或发现可弹出元素为止

# LREN

`LREM key count value`

* 时间复杂度 O(N), N 为列表的长度
* 移除列表中值为 value 的元素, count 的绝对值为移除元素的数量
* count = 0, 移除列表中所有值为 value 的元素
* count > 0, 从表头开始搜索并移除列表中为 value 的元素, count 为需要移除元素的数量
* count < 0, 从队尾开始搜索并移除列表中为 value 的元素, count 的绝对值为需要移除元素的数量
* 需要移除元素的数量, 如列表中有 5 个 a 元素, 当 count 或者绝对值为 4 时, 将会有 4 个 a 元素被移除
* 返回值为被成功移除元素的数量, 当列表不存在时, 返回 0

```java
// 下例中的 list 元素被移除后值为 [a+, a, b]
redisCommands.rpush("list", "a+", "a", "a", "a", "b");
redisCommands.lrem("list", 2, "a");
log.info("list = {}", redisCommands.lrange("list", 0, -1));
redisCommands.del("list");
```

# LINDEX

`LINDEX key index`

* 时间复杂度 O(N), N 为到达下标 index 过程中经过的元素数量, 对列表的头元素和尾元素执行 LINDEX 命令, 复杂度为O(1)
* index 的值可以为负数, -1 表示队列的最后一个元素
* 返回对应列表中下标对应的值, 当下标值大于列表长度时, 返回 null
* 如果 key 不存在或者 key 不是列表类型抛出异常

```java
// 下例返回 a+
redisCommands.rpush("list", "a+", "a", "a", "a", "b");
log.info("value = {}", redisCommands.lindex("list", 0));
redisCommands.del("list");
```

# LINSET

`LINSERT key BEFORE|AFTER indexValue insertValue`

* 时间复杂度 O(N), N 为值 indexValue 的所在下标的位置 (列表中有多个 indexValue 只会搜索到的第一个元素位置)
* 在列表中从左往右搜索值 indexValue, 在这个值的位置插入值 insertValue
* BEFORE 表示在搜索到的值之前插入值, AFTER 表示在搜索到的值之后插入值
* 返回值列表的长度, 若 indexValue 不存在或者列表不存在返回 0, 且不会插入新的值

```java
// 下例 list 在插入新值后值为 [1, 2, N, 3, 3, 4, 3]
redisCommands.rpush("list", "1", "2", "3", "3", "4", "3");
redisCommands.linsert("list", true, "3", "N");
log.info("list = {}", redisCommands.lrange("list", 0, -1));
redisCommands.del("list");
```

# LSET

`LSET key index value`

* 时间复杂度 O(N), 对于表头和队尾为 O(1)
* 设置列表对应位置的值, 如果队列不存在, 或者 index 的值大于列表时返回错误
* 成功返回 ok, 失败返回错误信息

```java
// 下例在设置值后 list 的值为 [1, 2, 3, 4]
redisCommands.rpush("list", "1", "2", "3", "5");
redisCommands.lset("list", 3, "4");
log.info("list = {}", redisCommands.lrange("list", 0, -1));
redisCommands.del("list");
```

# LTRIM

`LTRIM key start stop`

* 参考资料: [http://redisdoc.com/list/ltrim.html](http://redisdoc.com/list/ltrim.html)
* 列表只保留指定区间内的元素, 不在指定区间之内的元素都将被删除
* 时间复杂度 O(N), N 为被移除的元素的数量

```java
// 下例在删除元素后的 list 值为 [2, 3]
redisCommands.rpush("list", "1", "2", "3", "4");
redisCommands.ltrim("list", 1, 2);
log.info("list = {}", redisCommands.lrange("list", 0, -1));
redisCommands.del("list");
```
