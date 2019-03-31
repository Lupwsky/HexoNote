---
title: Redis-Redis哈希表
date: 2019-02-16 21:38:06
categories: Redis
---

# HSET 设置值

* HSET key field value
* 成功设置返回 1, 已经存在并且覆盖旧值返回0, 失败返回错误

```java
redisCommands.hset("user:TF001", "name", "lpw");
redisCommands.hset("user:TF001", "age", "18");
redisCommands.hset("user:TF001", "sex", "男");
```

# HSETNX 设置值

* HSETNX key field value
* 当 key 对应的 Hash 表不存在 field 时, 设置这个值, 如果存在则不做任何操作
* 成功时返回 1, 失败或者放弃操作返回 0

```java
boolean hsetnxResult = redisCommands.hsetnx("user:TF001", "name", "LPW");
log.info("hsetnxResult = {}", hsetnxResult);
```

<!-- more -->

# HMSET 批量设置值

* HMSET key field value [field value …]
* 批量设置 key 对应的 Hash 表中的值, 如果已经存在, 则覆盖, 设置成功返回 OK, 设置失败返回错误

```java
HashMap<String, String> hashMap = new HashMap<>();
hashMap.put("name", "LPW");
hashMap.put("age", "20");
String hmsetResult = redisCommands.hmset("user:TF001", hashMap);
log.info("hmsetResult = {}", hmsetResult);
```

# HGET 获取值

* HGET hash field
* 返回 Hash 表中指定 field 的 value, 对于不存在的 field 返回 null

```java
redisCommands.hget("user:TF001", "name");
```

# HMGET 批量获取值

* HMGET key field [field …]
* 批量获取 Hash 表指定的字段

```java
List<KeyValue<String, String>> keyValueList = redisCommands.hmget("user:TF001", "name", "age", "none");
KeyValue<String, String> keyValue = keyValueList.get(0);
log.info("key = {}, value = {}", keyValue.getKey(), keyValue.getValue());
keyValue = keyValueList.get(2);
log.info("key = {}, value = {}", keyValue.getKey(), keyValue.getValueOrElse(""));
```

# HGETALL 获取所有值

* HGETALL key
* 返回 key 对应 Hash 表中所有的键值对

```java
Map<String, String> resultMap = redisCommands.hgetall("user:TF001");
log.info("hset = {}", resultMap.toString());
```

# HEXISTS 检测 FIELD

* HEXISTS key field
* 检测 key 对应的 Hash 表中 field 是否存在, 存在返回 1, 不存在返回 0

```java
redisCommands.hexists("user:TF001", "name");
```

# HDEL 删除值

* HDEL key field [field …]
* 删除 key 对应的 Hash 表中对应的 field, 返回成功删除的数量

```java
redisCommands.hdel("user:TF001", "name");
redisCommands.hset("user:TF001", "name", "lpw");
```

# HKEYS 获取所有 FIELD

* HKEYS key
* 返回 key 对应的 Hash 表中所有的 field, 没有任何 field 或者 key 不存在返回一个空表

```java
List<String> keyList = redisCommands.hkeys("user:TF001");
log.info("keyList = {}", keyList.toString());
```

# HVALS 获取所有值

* HVALS key
* 返回 key 对应的 Hash 表中所有的值, 没有任何 field 或者 key 不存在返回一个空表

```java
List<String> valueList = redisCommands.hvals("user:TF001");
log.info("valueList = {}", valueList.toString());
```

# HLEN 获取 FIELD 数量

* HLEN key
* key 对应的 Hash 表中 field 的数量

```java
redisCommands.hlen("user:TF001");
```

# HSTRLEN 获取 FIELD 对应值的字符串长度

* HSTRLEN key field
* 返回 key 对应的 Hash 表中所有的 field 对应值的字符串长度

```java
redisCommands.hstrlen("user:TF001", "name");
```

# HINCRBY 和 HINCRBYFLOAT 对值加减

* HINCRBY key field increment - 加减整型数据
* HINCRBYFLOAT key field increment - 加减浮点型数据
* key 对应的 Hash 表中 field 的字段做加减操作, 如果 filed 对应的值如果是字符串, 会出现错误

```java
redisCommands.hincrby("user:TF001", "age", 2);
```