---
title: Java-Java8中的Optional类
date: 2018-10-29 22:19:00
categories:  Java
---

Optional 是个容, 它可以保存类型 T 的值，或者仅仅保存 null, Optional提供很多有用的方法，这样我们就不用显式进行空值检测, Optional 类的引入很好的解决空指针异常

# isPresent 方法

isPresent 用于判断对象的实例是否存在, true 表示存在, false 表示不存在, 即值为 null, 使用示例如下:

```java
Optional<DataNode> dataNodeOptional = Optional.empty();
if (dataNodeOptional.isPresent()) {
    log.info("[Optional isPresent 方法] dataNode = {}", dataNodeOptional.get().toString());
} else {
    log.info("[Optional isPresent 方法] dataNode 的值为 null");
}
```

<!-- more -->

# ifPresent 方法

ifPresent 如果实例存在, 执行其他的操作, 这里调用将会没有任何的输出, 使用示例如下:

```java
Optional<DataNode> dataNodeOptional = Optional.empty();
dataNodeOptional.ifPresent(data -> log.info("[Optional isPresent 方法] dataNode = {}", data.toString()));
```

# orElse 方法

orElse 方法, 如果为 null 就返回一个默认的对象实例, 使用示例如下:

```java
Optional<DataNode> dataNodeOptional = Optional.empty();
DataNode dataNode = dataNodeOptional.orElse(DataNode.builder().name("DEFAULT").soc(1).build());
log.info("[Optional orElse 方法] dataNode = {}", dataNode.toString());
```

# orElseGet 方法

orElseGet 方法, 和 orElse 方法一样, 如果为 null 就返回一个默认的对象实例, 只不过 orElseGet 方法接收的是一个 Supplier 接口的实例, 是一个 Lambda 表达式, 我们可以这样使用 return dataNode.orElseGet(() -> getFromDatabase()), 使用示例如下:

```java
Optional<DataNode> dataNodeOptional = Optional.empty();
DataNode dataNode = dataNodeOptional.orElseGet(() -> DataNode.builder().name("DEFAULT").soc(1).build());
log.info("[Optional orElseGet 方法] dataNode = {}", dataNode.toString());
```

# orElseThrow 方法

orElseThrow 如果为 null 就抛出一个异常, 使用示例如下:

```java
Optional<DataNode> dataNodeOptional = Optional.empty();
try {
    DataNode dataNode = dataNodeOptional.orElseThrow(() -> new IllegalArgumentException("对象的值为 null"));
    log.info("[Optional orElseThrow 方法] dataNode = {}", dataNode.toString());
} catch (Exception e) {
    log.error("[Optional orElseThrow 方法] error = {}", e.getMessage());
}
```

# map 和 filter 的方法

map 和 filter 的方法的使用在 Stream API 里面介绍过, 这里不详细介绍了, 使用示例如下:

```java
Optional<DataNode> dataNodeOptional = Optional.of(DataNode.builder().name("DEFAULT").soc(1).build());
Optional<String> nameOptional = dataNodeOptional.map((DataNode::getName));
log.info("[Optional map 方法] name = {}", nameOptional.orElse(""));
```

# 输出结果

上面代码的输出结果如下:

```text
22:43:47.644 [main] INFO com.thread.excutor.OptionalMain - [Optional isPresent 方法] dataNode 的值为 null
22:43:47.728 [main] INFO com.thread.excutor.OptionalMain - [Optional orElse 方法] dataNode = DataNode(name=DEFAULT, soc=1)
22:43:47.731 [main] INFO com.thread.excutor.OptionalMain - [Optional orElseGet 方法] dataNode = DataNode(name=DEFAULT, soc=1)
22:43:47.731 [main] ERROR com.thread.excutor.OptionalMain- [Optional orElseThrow 方法] error = 对象的值为 null
22:43:47.732 [main] INFO com.thread.excutor.OptionalMain - [Optional map 方法] name = DEFAULT
```

# 参考资料

* Java 8 实战 (书籍)
* [Java 8 Optional 的正确姿势](https://blog.csdn.net/hj7jay/article/details/52459334?utm_source=blogxgwz0)