---
title: Java-ArrayList和LinkedList循环遍历性能测试
date: 2018-10-20 00:43:21
categories: Java
---

# 集合数据在内存中的存储方式

Java 对于集合数据的遍历提供了几种不同的方式, 普通的 for 循环, 迭代器 Iterator 和 forEach, 在了解集合数据的遍历之前, 有必要先了解下集合数据在内存中如何存放方式, 主要有以下顺序存储和链式存储两种

## 顺序存储

顺序存储时保存数据的内存是连续的, 可以根据元素的位置直接计算出内存地址, 可以很方便的获取对应位置的元素, 但是顺序存储在删除和插入数据时不方便, 删除数据和插入数据时需要将删除元素后的所有元素都要往前或者往后移动, 在内存中使用这种存储方式的数据集合最具代表性的是 ArrayList

## 链式存储

链式存储保存数据时各个元素的内存不是连续的, 不能通过元素的位置计算出内存地址, 每个元素都会保存下一个元素的内存地址, 因此删除和插入数据很方便, 但是查找数据需要循环遍历才能找到 (原因时不能通过元素的位置计算出内存地址), 在内存中使用这种存储方式的数据集合最具代表性的是 LinkedList

<!-- more -->

## 一般性结论

了解上面的知识后, 可以大致得出一个一般性结论, 使用顺序存储的集合数据查询效率要高于链式存储, 而使用链式存储的集合数据在删除和插入的性能上要高于链式存储, `为什么是一般性结论, 因为决定这些效率的时要看查询, 删除和删除元素的在集合数据中的位置是相关的`

# 开始测试

下面的测试都是基于 ArrayList 和 LinkedList 来测试, 测试环境 Windows 10, i7-7770 CPU @ 3.60GHz, 16G RAM

## for 循环

for 循环时最基础也是 Java 最开始提供的循环方式, 适合于遍历顺序存储的集合数据, 测试代码如下:

```java
public static void main(String[] args) {
    // 模拟 10w 数据
    List<DataNode> dataNodeList1 = new ArrayList<>();
    List<DataNode> dataNodeList2 = new LinkedList<>();
    for (int i = 0; i < 1000000; i++) {
        dataNodeList1.add(DataNode.builder().soc(i).build());
        dataNodeList2.add(DataNode.builder().soc(i).build());
    }

    // ArrayList 测试
    DateTime startTime = DateTime.now();
    for (int i = 0; i < dataNodeList1.size(); i++) {
        dataNodeList1.get(i).setSoc(10000);
    }
    DateTime endTime = DateTime.now();
    log.info("[循环测试] ArrayList 的 for 循环, 花费时间 = {} ms",
        endTime.getMillis() - startTime.getMillis());

    // LinkedList 测试
    startTime = DateTime.now();
    for (int i = 0; i < dataNodeList2.size(); i++) {
        dataNodeList2.get(i).setSoc(10000);
    }
    endTime = DateTime.now();
    log.info("[循环测试], LinkedList 的 for 循环, 花费时间 = {} ms",
        endTime.getMillis() - startTime.getMillis());
}
```

输出结果如下:

```txt
[循环测试] ArrayList 的 for 循环,  花费时间 = 34 ms
[循环测试] LinkedList 的 for 循环, 花费时间 = 8441 ms

// 100w 数据, LinkedList 的循环我等了近一个小时仍然没有遍历完
[循环测试] ArrayList 的 for 循环,  花费时间 = 42 ms
```

可以看到 LinkedList 的效率是远低于 ArrayList 的, 主要是在 LinkedList 的 get 方法, LinkedList 需要不断的循环遍历获取值, 导致获取值耗时较多, 具体分析见 [Java-LinkedList源码分析]() 一节的笔记

## Iterator 迭代器循环

迭代器是对 Collection 进行迭代的类, 用于循环取出集合元素并对元素进行操作, Collection 接口中定义了获取集合类迭代器的方法 iterator(),  所有的 Collection 体系集合都可以获取自身的迭代器, 使用迭代器可以在循环时添加, 删除和替换元素, 不需要和 for 循环一样维护一个显示的计数器, 当然缺点也是由于没有显示的计数器, 在需要获取元素下标的情况下, 需要自己维护一个计数器, 这里测试下在相同的条件下, 循环遍历 ArrayList 和 LinkedList 的效率, 测试代码如下:

```java
DateTime startTime = DateTime.now();
DataNode dataNode;
Iterator<DataNode> dataNodeIterator1 = dataNodeList1.iterator();
while (dataNodeIterator1.hasNext()) {
    dataNode = dataNodeIterator1.next();
    dataNode.setSoc(10000);
}
DateTime endTime = DateTime.now();
log.info("[循环测试] ArrayList 的 Iterator 方式遍历, 花费时间 = {} ms",
    endTime.getMillis() - startTime.getMillis());

startTime = DateTime.now();
DataNode dataNode2;
Iterator<DataNode> dataNodeIterator2 = dataNodeList1.iterator();
while (dataNodeIterator2.hasNext()) {
    dataNode2 = dataNodeIterator2.next();
    dataNode2.setSoc(10000);
}
endTime = DateTime.now();
log.info("[循环测试] LinkedList 的 Iterator 方式遍历, 花费时间 = {} ms",
    endTime.getMillis() - startTime.getMillis());
```

输出结果如下:

```txt
// 10w 数据
[循环测试] ArrayList 的 Iterator 方式遍历, 花费时间 = 40 ms
[循环测试] LinkedList 的 Iterator 方式遍历, 花费时间 = 2 ms

// 100w 数据
[循环测试] ArrayList 的 Iterator 方式遍历, 花费时间 = 45 ms
[循环测试] LinkedList 的 Iterator 方式遍历, 花费时间 = 12 ms
```

使用迭代器的方式遍历, 很明显可以看到 LinkedList 的查询效率是得到明显的提升的, 具体见 [Java-ArrayList和LinkedList的迭代器源码分析]()

## forEach 循环

forEach 循环本质上使用的也是迭代器, 因此循环速度也是很快的, 相比直接使用迭代器, forEach 语句的代码简洁, 且遍历速度和直接使用迭代差别微小, 缺点就是不能在迭代器循环时对数据进行添加, 删除, 替换元素的操作, 测试代码如下:

```java
public static void main(String[] args) {
    List<DataNode> dataNodeList1 = new ArrayList<>();
    List<DataNode> dataNodeList2 = new LinkedList<>();
    for (int i = 0; i < 1000000; i++) {
        dataNodeList1.add(DataNode.builder().soc(i).build());
        dataNodeList2.add(DataNode.builder().soc(i).build());
    }

    DateTime startTime = DateTime.now();
    for (DataNode dataNode : dataNodeList1) {
        dataNode.setSoc(10000);
    }
    DateTime endTime = DateTime.now();
    log.info("[循环测试] ArrayList 的 forEach 方式遍历, 花费时间 = {} ms",
        endTime.getMillis() - startTime.getMillis());

    startTime = DateTime.now();
    for (DataNode dataNode : dataNodeList2) {
        dataNode.setSoc(10000);
    }
    endTime = DateTime.now();
    log.info("[循环测试] LinkedList 的 forEach 方式遍历, 花费时间 = {} ms",
        endTime.getMillis() - startTime.getMillis());
}
```

输出结果如下:

```txt
// 100w 数据
[循环测试] ArrayList 的 forEach 方式遍历, 花费时间 = 66 ms
[循环测试] LinkedList 的 forEach 方式遍历, 花费时间 = 16 ms
```

在 Java 8 后 forEach 还有另外一种调用的方法, 使用示例如下:

```java
DateTime startTime = DateTime.now();
dataNodeList1.forEach(dataNode -> dataNode.setSoc(10000));
DateTime endTime = DateTime.now();
log.info("[循环测试] ArrayList Java 8 中的 forEach 方式遍历, 花费时间 = {} ms",
    endTime.getMillis() - startTime.getMillis());

startTime = DateTime.now();
dataNodeList2.forEach(dataNode -> dataNode.setSoc(10000));
endTime = DateTime.now();
log.info("[循环测试] LinkedList Java 8 中的 forEach 方式遍历, 花费时间 = {} ms",
    endTime.getMillis() - startTime.getMillis());
```

输出结果如下:

```txt
// 100w 数据
[循环测试] ArrayList Java 8 中的 forEach 方式遍历, 花费时间 = 78 ms
[循环测试] LinkedList Java 8 中的 forEach 方式遍历, 花费时间 = 16 ms
```
