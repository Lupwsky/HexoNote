---
title: Java-Guava工具类之集合的使用
date: 2019-01-26 12:03:49
categories: Java
---

之前一直有听说过谷歌的 Guava, 但是在项目中也仅是非常简单使用过其中的缓存功能 , 称着这段时间项目不是很忙, 抓紧时间系统的了解下 (主要参考 [官方的文档](https://github.com/google/guava) 和 [Guava官方教程(中文版)](https://willnewii.gitbooks.io/google-guava/content/index.html)), 这里记录下自己的学习笔记, 这篇笔记主要是关于 Guava 中提供了一些新的集合类型的

# MultiSet

MultiSet 是 Set 的延伸, MultiSet 是继承 Collection 实现的, 它可以分成两种方式来使用, 将 MultiSet 看做一个普通的  Set 时, 是一个允许出现重复的元素 Set , 因此和 JDK 自带的 Set 不同, MultiSet 允许出现重复的元素, 当看成 Map<E, Integer> 类型时, 则表示键为元素, 值为计数的 Map 类型, 可以方便的对键的值进行计数

## MultiSet 作为 Set 使用允许元素重复出现

```java
Set<String> set = new HashSet<>();
set.add("1");
set.add("2");
set.add("2");
set.add("3");
set.forEach(log::info);
```

<!-- more -->

输出结果如下:

```text
12:08:26.686 [main] INFO com.lupw.guava.set.GuavaSetMain - 1
12:08:26.691 [main] INFO com.lupw.guava.set.GuavaSetMain - 2
12:08:26.691 [main] INFO com.lupw.guava.set.GuavaSetMain - 3
```

MultiSet 运行重复的元素出现, 示例如下:

```java
Multiset<String> multiset = HashMultiset.create();
multiset.add("1");
multiset.add("2");
multiset.add("2");
multiset.add("3");
multiset.forEach(log::info);
```

输出结果如下:

```text
12:08:26.709 [main] INFO com.lupw.guava.set.GuavaSetMain - 1
12:08:26.709 [main] INFO com.lupw.guava.set.GuavaSetMain - 2
12:08:26.709 [main] INFO com.lupw.guava.set.GuavaSetMain - 2
12:08:26.709 [main] INFO com.lupw.guava.set.GuavaSetMain - 3
```

## MultiSet 作为 Map 使用对键的值计数

在 JDK 中使用 Map 计数的实现方式

```java
// 在 JDK 7 之前Map 实现计数
Map<String, Integer> countsMap = new HashMap<>();
if (countsMap.containsKey("1")) {
    int count = countsMap.get("1");
    countsMap.put("1", count + 1);
} else {
    countsMap.put("1", 0);
}
```

使用 MultiSet 进行计数:

```java
Multiset<String> multiset = HashMultiset.create();
multiset.add("1");
// key 为 1 的键加两次计数, 可以理解为是两个 add("1")
multiset.add("1", 2);
log.info(String.valueOf(multiset.count("1")));
```

输出结果如下:

```text
12:08:26.718 [main] INFO com.lupw.guava.set.GuavaSetMain - 3
```

## MultiSet 看作 Map 时对应 JDK 中的类

将 MultiSet 用作 Map 时, 对应 JDK 各个 Map 的实现如下

| JDK | Guava |
|---|---|
| HashMap | HashMultiset |
| TreeMap | TreeMultiset |
| LinkedHashMap | LinkedHashMultiset |
| LinkedHashMultiset | ConcurrentHashMultiset |
| ImmutableMap | ImmutableMultiset |

## 一些 MultiSet 的其他方法

| 方法 | 作用 |
|---|---|
| iterator() | 返回一个迭代器, 包含 Multiset 的所有元素 |
| size() | 返回包括重复的元素所有元素的总计数数量 |
| count() | 返回给定元素的计数 |
| elementSet() | 返回所有不重复元素的 Set, elementSet().size() 可以获取到不重复的元素数量 |
| entrySet() | 返回包括重复元素的 Set<Multiset.Entry> |
| remove(E e, int count) | 设置给定元素的计数, 不允许为负数, 为 0 时表示删除元素 |
| setCount(E e, int count) | HashMultiset |
| setCount(E e, int oldCount, int newCount) | 设置给定元素的计数, 如果 oldCount 的值和目前计数相同, 则设置成新值, 否则不变 |

# Multimap

## Multimap 解决多键值索引的数据结构

Multimap 主要被设置为解决于多键值索引的数据结构的场景, 例如使用 Map 实现这样的数据结构, 我们必须要定义成 `Map<String, List<Object>> ObjectMap = new HashMap<>>()` 这样的类型, 在使用的时候需要检查 key 是否存在, 不存在时则创建一个 List, 将值放入 List 后在放入 Map, 存在时在先取出 List , 然后再 List 后面添加后再将 List 放入 Map, 有过这样开发经验的人都回知道这样的过程是比较繁琐的, Multimap 可以很方便的解决这个问题, Guaua 提供常用有继承于 ListMultimap 和 SetMultimap 的子类, 如 ArrayListMultimap, 使用示例如下:

```java
// ArrayListMultimap<String, String> -> Map<String, List<String>>
ArrayListMultimap<String, String> multimap = ArrayListMultimap.create();
// 如果 对应键值的 List 不存在, put 的时候会自动创建, 免去我们还要去检测的步骤
multimap.put("1", "1");
List<String> dataList1 = multimap.get("1");
dataList1.forEach(log::info);

// 如果获取的 key 对应的 List 不存在, 会返回一个空的集合
List<String> dataList2 = multimap.get("2");
log.info("{}", dataList2.size());
```

## Guava 提供的 Multimap 类

| 类名 |
|---|
| ArrayListMultimap |
| HashMultimap |
| LinkedListMultimap |
| LinkedHashMultimap |
| TreeMultimap |
| ImmutableListMultimap |
| ImmutableSetMultimap |

# 不可变集合

不可变集合在 JDK 中也有提供, 使用 Collections.unmodifiableSet() 方法将原集合包装为不可变集合, 但是 JDK 不可变集合对原集合的引用进行修改, 仍然可以更改元素, 这种不可变需要保证原始集合没有人引用, 这里主要选 JDK 的 Set 包装成不可变集合和 ImmutableMultiSet 进行一下比较, 其他类型就没有去测试了

```java
Set<String> set = new HashSet<>();
set.add("1");
set.add("2");
set.add("3");
set.add("4");

// Collections.unmodifiableSet() 方法将原集合包装为不可变集合
Set<String> unmodifiableSet = Collections.unmodifiableSet(set);
unmodifiableSet.forEach(log::info);

// JDK 不可变集合对原集合的引用进行修改, 仍然可以更改元素, 这种不可变需要保证原始集合没有人引用
set.add("6");
unmodifiableSet.forEach(log::info);

// JDK 不可变集合虽然可以调用 add 方法, 但是会出现 UnsupportedOperationException 异常, 方法没有被屏蔽
unmodifiableSet.add("7");
unmodifiableSet.forEach(log::info);

// Guava 不可变集合的使用
ImmutableMultiset<String> immutableMultiset = ImmutableMultiset.of("1", "2", "2", "3", "4");
```

输出结果如下:

```text
13:06:58.625 [main] INFO com.lupw.guava.set.GuavaSetMain - 1
13:06:58.629 [main] INFO com.lupw.guava.set.GuavaSetMain - 2
13:06:58.629 [main] INFO com.lupw.guava.set.GuavaSetMain - 3
13:06:58.629 [main] INFO com.lupw.guava.set.GuavaSetMain - 4

# 不可变集合对原集合的引用进行修改后成功将元素 6 添加进去
13:06:58.630 [main] INFO com.lupw.guava.set.GuavaSetMain - 1
13:06:58.630 [main] INFO com.lupw.guava.set.GuavaSetMain - 2
13:06:58.630 [main] INFO com.lupw.guava.set.GuavaSetMain - 3
13:06:58.630 [main] INFO com.lupw.guava.set.GuavaSetMain - 4
13:06:58.630 [main] INFO com.lupw.guava.set.GuavaSetMain - 6
```

# BiMap

## BiMap 双向映射

BiMap 主要被设计于解决双向映射的, 在 JDK 中通常的做法需要维护两张表, 同步的时候也需要特别小心, 两边都要同步, 如果有一个没有同步, 很有可能造成程序上的逻辑异常, 如下:

```java
Map<String, Integer> nameToIdMap = Maps.newHashMap();
Map<Integer, String> idToNameMap = Maps.newHashMap();
nameToIdMap.put("lpw", 1);
idToNameMap.put(1, "lpw");
```

BiMap 核心的方法为 inverse(), 对 Map 进行反转后可以获取反转后的 Map, 对反转后的 Map 进行修改会同步到原 Map, 对原 Map 进行修改也会同步到反转后的 Map, 使用示例如下:

```java
// 此时 biMap 里面有 key1 = value1, key2 = value2 两组值
BiMap<String, String> biMap = HashBiMap.create();
biMap.put("key1", "value1");
biMap.put("key2", "value2");
log.info(biMap.get("key1"));
log.info(biMap.get("key2"));

// 对于已经存在的值, 不允许使用新的键再次进行映射, 出现 IllegalArgumentException 异常
try {
    biMap.put("key3", "value1");
} catch (IllegalArgumentException e){
    log.info("error = {}", e);
}

// 如果有需要, 使用 forcePut 方法可以强制替换值对应的键, 重新进行映射, 此时有 key3 = value1, key2 = value2 两组值
biMap.forcePut("key3", "value1");
log.info(biMap.toString());

// inverse() 反转 BiMap<K, V> 的键值映射
log.info(biMap.get("key3"));

// 此时有 value1 = key3, value2 = key2 两组值
BiMap<String, String> inverseBiMap = biMap.inverse();
log.info(inverseBiMap.get("value1"));

// 对 biMap 添加新的键值对或者进行修改, 会直接同步到inverseBiMap
// 此时 biMap 有 key3 = value1, key2 = value2, key4 = value4
// 此时 inverseBiMap 有 value1 = key3, value2 = key2, value4 = key4
// 同样对反转后的 inverseBiMap 进行修改也会直接同步到 biMap
biMap.put("key4", "value4");
biMap.forEach(log::info);
inverseBiMap.forEach(log::info);
```

## Guava 提供的 BiMap 类

| 类名 |
|---|
| HashBiMap |
| ImmutableBiMap |
| EnumBiMap |
| EnumHashBiMap |

# Table

通常我们选择使用诸如 Map<String, Map<String, String>> 这样的数据结构类型来实现多个键做索引获取值的情况, 通过第一个 key 和第二个 key 一同确认最终需要的值, 无论是获取值和存入值都是不便于处理的, Guava 提供了新的集合类型 Table 来实现多个键做索引, 可以友好的处理多个键做索引获取值的需求

## HashBasedTable 的使用

```java
Table<String, String, String> hashBasedTable = HashBasedTable.create();
hashBasedTable.put("R1", "C1", "R1C1");
hashBasedTable.put("R1", "C2", "R1C2");
hashBasedTable.put("R1", "C3", "R1C3");
hashBasedTable.put("R2", "C1", "R2C1");
hashBasedTable.put("R2", "C2", "R2C2");
hashBasedTable.put("R2", "C3", "R2C3");
hashBasedTable.put("R3", "C1", "R3C1");
hashBasedTable.put("R3", "C2", "R3C2");
hashBasedTable.put("R3", "C3", "R3C3");

log.info(hashBasedTable.get("R2", "C2"));
log.info("------------------------------------------------");

// rowMap() 方法将 Table 转换成 Map<T, Map<T, T>> 的数据结构类型, 第一个键值为行, 第二个键值为列, 对这个结果集的操作同样会同步到到 Table 中
Map<String, Map<String, String>> rowMapMap = hashBasedTable.rowMap();
rowMapMap.forEach((key, value) -> {
    log.info("key = {}, value = {}", key, value);
});
log.info("------------------------------------------------");

// rowMap() 方法将 Table 转换成 Map<T, Map<T, T>> 的数据结构类型, 第一个键值为列, 第二个键值为行, 对这个结果集的操作同样会同步到到 Table 中
Map<String, Map<String, String>> columnMapMap = hashBasedTable.columnMap();
columnMapMap.forEach((key, value) -> {
    log.info("key = {}, value = {}", key, value);
});
log.info("------------------------------------------------");

// row(key) 方法返回给定行的值集合, 数据结构类型为 Map<T, T>, 对这个结果集的操作同样会同步到到 Table 中
Map<String, String> columnMap = hashBasedTable.row("R1");
columnMap.forEach((key, value) -> {
    log.info("key = {}, value = {}", key, value);
});
log.info("------------------------------------------------");

// column(key) 方法返回给定列的值集合, 数据结构类型为 Map<T, T>, 对这个结果集的操作同样会同步到到 Table 中
Map<String, String> rowMap = hashBasedTable.column("C1");
rowMap.forEach((key, value) -> {
    log.info("key = {}, value = {}", key, value);
});
log.info("------------------------------------------------");

// cellSet() 获取集合
Set<Table.Cell<String, String, String>> cellSet = hashBasedTable.cellSet();
cellSet.forEach(cell -> {
    log.info("rowKey = {}, columnKey = {}, value = {}", cell.getRowKey(), cell.getColumnKey(), cell.getValue());
});
log.info("------------------------------------------------");
```

输出结果如下:

```text
13:19:44.899 [main] INFO com.lupw.guava.set.GuavaTableMain - R2C2
13:19:44.903 [main] INFO com.lupw.guava.set.GuavaTableMain - ------------------------------------------------
13:19:44.947 [main] INFO com.lupw.guava.set.GuavaTableMain - key = R1, value = {C1=R1C1, C2=R1C2, C3=R1C3}
13:19:44.951 [main] INFO com.lupw.guava.set.GuavaTableMain - key = R2, value = {C1=R2C1, C2=R2C2, C3=R2C3}
13:19:44.951 [main] INFO com.lupw.guava.set.GuavaTableMain - key = R3, value = {C1=R3C1, C2=R3C2, C3=R3C3}
13:19:44.951 [main] INFO com.lupw.guava.set.GuavaTableMain - ------------------------------------------------
13:19:44.969 [main] INFO com.lupw.guava.set.GuavaTableMain - key = C1, value = {R1=R1C1, R2=R2C1, R3=R3C1}
13:19:44.971 [main] INFO com.lupw.guava.set.GuavaTableMain - key = C2, value = {R1=R1C2, R2=R2C2, R3=R3C2}
13:19:44.971 [main] INFO com.lupw.guava.set.GuavaTableMain - key = C3, value = {R1=R1C3, R2=R2C3, R3=R3C3}
13:19:44.971 [main] INFO com.lupw.guava.set.GuavaTableMain - ------------------------------------------------
13:19:44.971 [main] INFO com.lupw.guava.set.GuavaTableMain - key = C1, value = R1C1
13:19:44.971 [main] INFO com.lupw.guava.set.GuavaTableMain - key = C2, value = R1C2
13:19:44.971 [main] INFO com.lupw.guava.set.GuavaTableMain - key = C3, value = R1C3
13:19:44.971 [main] INFO com.lupw.guava.set.GuavaTableMain - ------------------------------------------------
13:19:44.972 [main] INFO com.lupw.guava.set.GuavaTableMain - key = R1, value = R1C1
13:19:44.972 [main] INFO com.lupw.guava.set.GuavaTableMain - key = R2, value = R2C1
13:19:44.972 [main] INFO com.lupw.guava.set.GuavaTableMain - key = R3, value = R3C1
13:19:44.972 [main] INFO com.lupw.guava.set.GuavaTableMain - ------------------------------------------------
Disconnected from the target VM, address: '127.0.0.1:56436', transport: 'socket'
13:19:44.976 [main] INFO com.lupw.guava.set.GuavaTableMain - rowKey = R1, columnKey = C1, value = R1C1
13:19:44.976 [main] INFO com.lupw.guava.set.GuavaTableMain - rowKey = R1, columnKey = C2, value = R1C2
13:19:44.976 [main] INFO com.lupw.guava.set.GuavaTableMain - rowKey = R1, columnKey = C3, value = R1C3
13:19:44.976 [main] INFO com.lupw.guava.set.GuavaTableMain - rowKey = R2, columnKey = C1, value = R2C1
13:19:44.976 [main] INFO com.lupw.guava.set.GuavaTableMain - rowKey = R2, columnKey = C2, value = R2C2
13:19:44.976 [main] INFO com.lupw.guava.set.GuavaTableMain - rowKey = R2, columnKey = C3, value = R2C3
13:19:44.976 [main] INFO com.lupw.guava.set.GuavaTableMain - rowKey = R3, columnKey = C1, value = R3C1
13:19:44.976 [main] INFO com.lupw.guava.set.GuavaTableMain - rowKey = R3, columnKey = C2, value = R3C2
13:19:44.976 [main] INFO com.lupw.guava.set.GuavaTableMain - rowKey = R3, columnKey = C3, value = R3C3
13:19:44.976 [main] INFO com.lupw.guava.set.GuavaTableMain - ------------------------------------------------
```

## Guava 提供的 Table 类

| 类名 |
|---|
| HashBasedTable, 本质上用 HashMap<R, HashMap<C, V>> 实现 |
| ImmuTreeBasedTable, 本质上用 TreeMap<R, TreeMap<C,V>> 实现tableBiMap |
| ImmutableTable, 本质上用 ImmutableMap<R, ImmutableMap<C, V>> 实现, 注: ImmutableTable 对稀疏或密集的数据集都有优化 |
| ArrayTable, 要求在构造时就指定行和列的大小, 本质上由一个二维数组实现, 以提升访问速度和密集 Table 的内存利用率 |

# ClassToInstanceMap

ClassToInstanceMap 是一种特殊的 Map, 它的键是类型, 而值是符合键所指类型的对象对象, 用法和普通的 Map 没有区别, Guava 提供了两种有用的实现, MutableClassToInstanceMap 和 ImmutableClassToInstanceMap, 使用示例如下:

```java
ClassToInstanceMap<String> classToInstanceMap = MutableClassToInstanceMap.create();
classToInstanceMap.put(String.class, "1");
log.info(classToInstanceMap.get(String.class));

// 将值更改为 2
classToInstanceMap.put(String.class, "2");
log.info(classToInstanceMap.get(String.class));
```