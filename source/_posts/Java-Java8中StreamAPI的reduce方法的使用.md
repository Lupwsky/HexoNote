---
title: Java-Java8中StreamAPI的reduce方法的使用
date: 2018-10-17 23:38:14
categories:  Java
---

reduce 可以根据计算模型从 stream 中得到一个值, stream 中的 count 方法, max 方法, min 方法本质上都是 reduce 方法实现的, 由于这些方法常用, 因此在封装后放入了标准库中, reduce 一共有三个重载方法, 如下:

* `Optional<T> reduce(BinaryOperator<T> accumulator)`
* `T reduce(T identity, BinaryOperator<T> accumulator)`
* `<U> U reduce(U identity, BiFunction<U, ? super T, U> accumulator, BinaryOperator<U> combiner)`

# Optional<T> reduce(BinaryOperator<T> accumulator) 方法

该方法接受一个函数, 函数有两个参数, 第一个参数是计算后返回的值, 每次进行计算后的值会被设置到第一个参数里面, 如果是第一次计算, 这个值是 stream 里面的第一个元素, 而第二个参数是每次计算时 stream 对应的元素, 第一次计算是 stream 的第二个元素, 第二次则是 stream 的第三个元素, 以此类推 

<!-- more -->

使用示例如下:

```java
// DataNode 里面只有两个成员属性, 一个 String 类型的 name, 和一个 int 类型的 soc
List<DataNode> dataNodeList = new ArrayList<>();
dataNodeList.add(DataNode.builder().name("A").soc(1).build());
dataNodeList.add(DataNode.builder().name("B").soc(1).build());
dataNodeList.add(DataNode.builder().name("C").soc(1).build());
dataNodeList.add(DataNode.builder().name("D").soc(1).build());
dataNodeList.add(DataNode.builder().name("E").soc(1).build());

Optional<DataNode> optionalDataNode = dataNodeList.stream().reduce((result, item) -> {
    // 上一次计算返回的值, 如果是第一次计算, 这里是 stream 里面的第一个元素
    log.info("{} = {}", "result", JSON.toJSONString(result));

    // 当前计算使用的元素, 如果是第一次计算, 这里是 stream 里面的第二个元素, 第二次计算是 stream 的第三个元素, 以此类推
    log.info("{} = {}", "item  ", JSON.toJSONString(item));

    // 数据处理
    int total = result.getSoc() + item.getSoc();

    // 这里返回的值将会被设置到 result 中, 在下一次计算的时候使用
    return DataNode.builder().name(String.valueOf(total)).soc(total).build();
});

// orElse 方法, 如果 optionalDataNode 值为空就返回默认的 DataNode 值
DataNode dataNode = optionalDataNode.orElse(DataNode.builder().build());
log.info("finalDataNode" + JSON.toJSONString(dataNode));
```

输出结果如下:

```txt
[main] - result = {"name":"A","soc":1}
[main] - item   = {"name":"B","soc":1}

[main] - result = {"name":"2","soc":2}
[main] - item   = {"name":"C","soc":1}

[main] - result = {"name":"3","soc":3}
[main] - item   = {"name":"D","soc":1}

[main] - result = {"name":"4","soc":4}
[main] - item   = {"name":"E","soc":1}

[main] - finalDataNode{"name":"5","soc":5}
```

# T reduce(T identity, BinaryOperator<T> accumulator) 方法

该方法接受一个初始值和一个函数, 函数同样是有两个参数, 和 `Optional<T> reduce(BinaryOperator<T> accumulator)` 方法里面参数的作用是一样的, 但是由于有初始值, 第一次计算时, 第一个参数被设置的值是这个初始值, 第二个参数是 stream 里面的第一个元素, 第二次计算时, 第一个参数会被设置未第一次计算返回的值, 第二个参数则是 strea 里面的第二个元素, 之后以此类推, 同时如果 stream 为 null, 会将初始值返回 (不是 Optional 类型), 因此这个方法最终返回的值不会为 null

使用示例如下:

```java
DataNode initDataNode = DataNode.builder().name("I").soc(5).build();
DataNode finalDataNode = dataNodeList.stream().reduce(initDataNode, (result, item) -> {
    log.info("{} = {}", "result", JSON.toJSONString(result));
    log.info("{} = {}", "item  ", JSON.toJSONString(item));
    int total = result.getSoc() + item.getSoc();
    return DataNode.builder().name(String.valueOf(total)).soc(total).build();
});
log.info("finalDataNode" + JSON.toJSONString(finalDataNode));
```

输出结果如下:

```txt
[main] - result = {"name":"I","soc":5}
[main] - item   = {"name":"A","soc":1}

[main] - result = {"name":"6","soc":6}
[main] - item   = {"name":"B","soc":1}

[main] - result = {"name":"7","soc":7}
[main] - item   = {"name":"C","soc":1}

[main] - result = {"name":"8","soc":8}
[main] - item   = {"name":"D","soc":1}

[main] - result = {"name":"9","soc":9}
[main] - item   = {"name":"E","soc":1}

[main] - finalDataNode{"name":"10","soc":10}
```

# <U> U reduce(U identity, BiFunction<U, ? super T, U> accumulator, BinaryOperator<U> combiner) 方法

该方法接收三个参数, 最主要的是了解第三个参数的作用, 第三个参数的作用在于并发情况下`合并每个线程得到的最终结果`, 只有在并发情况下才会被调用

使用示例如下:

```java
List<DataNode> dataNodeList = new ArrayList<>();
dataNodeList.add(DataNode.builder().name("A").soc(1).build());
dataNodeList.add(DataNode.builder().name("B").soc(1).build());
dataNodeList.add(DataNode.builder().name("C").soc(1).build());
dataNodeList.add(DataNode.builder().name("D").soc(1).build());
dataNodeList.add(DataNode.builder().name("E").soc(1).build());
dataNodeList.add(DataNode.builder().name("F").soc(1).build());
dataNodeList.add(DataNode.builder().name("G").soc(1).build());
dataNodeList.add(DataNode.builder().name("H").soc(1).build());
dataNodeList.add(DataNode.builder().name("I").soc(1).build());
dataNodeList.add(DataNode.builder().name("J").soc(1).build());
dataNodeList.add(DataNode.builder().name("K").soc(1).build());
dataNodeList.add(DataNode.builder().name("L").soc(1).build());
dataNodeList.add(DataNode.builder().name("M").soc(1).build());
dataNodeList.add(DataNode.builder().name("N").soc(1).build());
dataNodeList.add(DataNode.builder().name("O").soc(1).build());
dataNodeList.add(DataNode.builder().name("P").soc(1).build());
dataNodeList.add(DataNode.builder().name("Q").soc(1).build());
dataNodeList.add(DataNode.builder().name("R").soc(1).build());
dataNodeList.add(DataNode.builder().name("S").soc(1).build());
dataNodeList.add(DataNode.builder().name("T").soc(1).build());
dataNodeList.add(DataNode.builder().name("U").soc(1).build());
dataNodeList.add(DataNode.builder().name("V").soc(1).build());
dataNodeList.add(DataNode.builder().name("W").soc(1).build());
dataNodeList.add(DataNode.builder().name("X").soc(1).build());
dataNodeList.add(DataNode.builder().name("Y").soc(1).build());
dataNodeList.add(DataNode.builder().name("Z").soc(1).build());

DataNode initDataNode = DataNode.builder().name("I").soc(5).build();

// 注意并发使用的 parallelStream 方法, 如果使用 stream 方法则不会调用第三个参数里面方法
DataNode finalDataNode = dataNodeList.parallelStream().reduce(initDataNode, (result, item) -> {
    log.info("{} = {}", "result", JSON.toJSONString(result));
    log.info("{} = {}", "item  ", JSON.toJSONString(item));
    int total = result.getSoc() + item.getSoc();
    return DataNode.builder().name("N").soc(total).build();
}, (result, item) -> {
    int total = result.getSoc() + item.getSoc();
    return DataNode.builder().name("F").soc(total).build();
});
log.info("finalDataNode" + JSON.toJSONString(finalDataNode));
```

输出结果如下:

```txt
[ForkJoinPool.commonPool-worker-1] - result = {"name":"I","soc":5}
[ForkJoinPool.commonPool-worker-5] - result = {"name":"I","soc":5}
[ForkJoinPool.commonPool-worker-3] - result = {"name":"I","soc":5}
[ForkJoinPool.commonPool-worker-7] - result = {"name":"I","soc":5}
[ForkJoinPool.commonPool-worker-7] - item   = {"name":"W","soc":1}
[ForkJoinPool.commonPool-worker-7] - result = {"name":"I","soc":5}
[ForkJoinPool.commonPool-worker-7] - item   = {"name":"L","soc":1}
[ForkJoinPool.commonPool-worker-1] - item   = {"name":"X","soc":1}
[main                            ] - result = {"name":"I","soc":5}
[ForkJoinPool.commonPool-worker-4] - result = {"name":"I","soc":5}
[main                            ] - item   = {"name":"Q","soc":1}
[ForkJoinPool.commonPool-worker-1] - result = {"name":"I","soc":5}
[ForkJoinPool.commonPool-worker-1] - item   = {"name":"B","soc":1}
[main                            ] - result = {"name":"I","soc":5}
[ForkJoinPool.commonPool-worker-1] - result = {"name":"I","soc":5}
[main                            ] - item   = {"name":"S","soc":1}
[ForkJoinPool.commonPool-worker-1] - item   = {"name":"C","soc":1}
[main                            ] - result = {"name":"I","soc":5}
[main                            ] - item   = {"name":"R","soc":1}
[ForkJoinPool.commonPool-worker-1] - result = {"name":"I","soc":5}
[main                            ] - result = {"name":"I","soc":5}
[main                            ] - item   = {"name":"O","soc":1}
[ForkJoinPool.commonPool-worker-1] - item   = {"name":"A","soc":1}
[main                            ] - result = {"name":"I","soc":5}
[ForkJoinPool.commonPool-worker-1] - result = {"name":"I","soc":5}
[main                            ] - item   = {"name":"P","soc":1}
[ForkJoinPool.commonPool-worker-1] - item   = {"name":"K","soc":1}
[main                            ] - result = {"name":"I","soc":5}
[ForkJoinPool.commonPool-worker-1] - result = {"name":"I","soc":5}
[ForkJoinPool.commonPool-worker-1] - item   = {"name":"J","soc":1}
[ForkJoinPool.commonPool-worker-1] - result = {"name":"I","soc":5}
[ForkJoinPool.commonPool-worker-1] - item   = {"name":"Y","soc":1}
[ForkJoinPool.commonPool-worker-1] - result = {"name":"I","soc":5}
[ForkJoinPool.commonPool-worker-1] - item   = {"name":"G","soc":1}
[ForkJoinPool.commonPool-worker-1] - result = {"name":"I","soc":5}
[ForkJoinPool.commonPool-worker-1] - item   = {"name":"I","soc":1}
[ForkJoinPool.commonPool-worker-1] - result = {"name":"I","soc":5}
[ForkJoinPool.commonPool-worker-6] - result = {"name":"I","soc":5}
[ForkJoinPool.commonPool-worker-1] - item   = {"name":"V","soc":1}
[ForkJoinPool.commonPool-worker-1] - result = {"name":"I","soc":5}
[ForkJoinPool.commonPool-worker-2] - result = {"name":"I","soc":5}
[ForkJoinPool.commonPool-worker-1] - item   = {"name":"F","soc":1}
[ForkJoinPool.commonPool-worker-5] - item   = {"name":"Z","soc":1}
[ForkJoinPool.commonPool-worker-2] - item   = {"name":"H","soc":1}
[ForkJoinPool.commonPool-worker-1] - result = {"name":"I","soc":5}
[ForkJoinPool.commonPool-worker-6] - item   = {"name":"T","soc":1}
[ForkJoinPool.commonPool-worker-3] - item   = {"name":"U","soc":1}
[ForkJoinPool.commonPool-worker-7] - result = {"name":"I","soc":5}
[ForkJoinPool.commonPool-worker-4] - item   = {"name":"D","soc":1}
[main                            ] - item   = {"name":"N","soc":1}
[ForkJoinPool.commonPool-worker-1] - item   = {"name":"E","soc":1}
[ForkJoinPool.commonPool-worker-7] - item   = {"name":"M","soc":1}
[main                            ] - finalDataNode{"name":"F","soc":156}
```

可以在输出日志看到, 调用了多线程来处理流, 每次调用第二个参数里面的方法时都使用了一个单独的线程处理 (线程池), 总共调用了 26 次, 每个线程计算的时候都会将初始化值带入计算, 一共 26 次, 因此最终结果为 `26 * 5 + 26 * 1 = 156`, 如果初始的值设置为 0 `(DataNode initDataNode = DataNode.builder().name("I").soc(0).build())`, 则最终的结果为 26 了

# 参考资料

* Java8 实战 (书籍)
* [Java 8 中 3 个参数的 reduce 方法怎么理解](https://segmentfault.com/q/1010000004944450/a-1020000007721483)
* [Java 8 系列之 Stream 中万能的 reduce](https://blog.csdn.net/IO_Field/article/details/54971679?utm_source=blogxgwz0)
