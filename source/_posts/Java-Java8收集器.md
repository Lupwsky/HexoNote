---
title: Java-Java8收集器
date: 2018-10-10 23:56:37
categories: Java
---

# 收集器

collect(Collectors.toList()) 将一个流转换成 List 的数据结构, 有时候需要生成其他值, 比如 Map 或者 Set, 或者生成自己定义的数据结构, 那么久必须使用收集器了, 收集器可以将流生成复杂的数据结构, 只需要将这个收集器传递到 collect 方法中。标准类库中使用了提供了一些收集器，如java.util.stream.Collectors类, 使用方法示例如下:

## 流转换成集合

```java
List<Map<String, Object>> mapList = dataList.stream()
        .map(x -> {
            HashMap<String, Object> map = new HashMap<>();
            map.put("index", x);
            return map;
        })
        .collect(Collectors.toList());
```

<!-- more -->

注意, 返回的是 List<Map<String, Object>> 对象, 但是调用 toList (也许是 toSet 等)方法时没有指定具体的类型, Stream 类库会自动推导合适的类型。如果需要自己指定类型，可以调用 toCollection 方法来指定, 示例代码如下:

```java
List<Map<String, Object>> mapList = dataList.stream()
        .map(x -> {
            HashMap<String, Object> map = new HashMap<>();
            map.put("index", x);
            return map;
        })
        .collect(Collectors.toCollection(ArrayList::new));
```

## 流转换成值

### maxBy、minBy 求最大最小值

maxBy、minBy求最大最小值, 需要传入一个比较器作为参数

```java
List<String> dataList = Arrays.asList("1", "2", "3", "4", "5", "6", "7");
Optional<String> a = dataList.stream().collect(Collectors.maxBy((o1, o2) -> Integer.valueOf(o1) - Integer.valueOf(o2)));
a.ifPresent(logger::debug);

// 等同于以下代码
if (a.isPresent()) {
    logger.debug(a.get());
}
```

### averagingInt 等求平均值

```java
List<Integer> dataList = Arrays.asList(1, 2, 3, 4, 5, 6, 7);
Double a = dataList.stream().collect(Collectors.averagingInt(x -> (int) x));
logger.debug(a);
```

### partitioningBy 将数据分块

```java
// 直接分区不进行任何的操作
List<Integer> dataList = Arrays.asList(1, 2, 3, 4, 5, 6, 7);
Map<Boolean, List<Integer>> map = dataList.stream().collect(Collectors.partitioningBy(x -> x % 2 == 1));
map.get(true).forEach(x -> System.out.print(x + " "));
System.out.println();
map.get(false).forEach(x -> System.out.print(x + " "));
System.out.println();
```

```txt
输出结果：
1 3 5 7
2 4 6 8 8
```

```java
// 可以在分区后再将流转换成值
List<Integer> dataList = Arrays.asList(1, 2, 3, 4, 5, 6, 7, 8, 8);
Map<Boolean, Double> map = dataList.stream().collect(Collectors.partitioningBy((x -> x % 2 == 1),
        Collectors.averagingInt(x -> (int) x)));
logger.debug(map.get(true));
logger.debug(map.get(false));
```

```txt
输出结果：
4.0
5.6
```

### groupingBy 将数据进行分组 (可以和分区联合使用)

```java
List<Integer> dataList = Arrays.asList(1, 4, 4, 4, 7, 8, 7, 8, 8);
Map<Integer, List<Integer>> map = dataList.stream().collect(Collectors.groupingBy(x -> x));
map.get(1).forEach(x -> System.out.print(x + " "));
System.out.println();
map.get(4).forEach(x -> System.out.print(x + " "));
System.out.println();
map.get(7).forEach(x -> System.out.print(x + " "));
System.out.println();
map.get(8).forEach(x -> System.out.print(x + " "));
System.out.println();
```

```txt
输出结果：
1
4 4 4
7 7
8 8 8
```

### counting 将数据进行统计

```java
Map<Integer, Long> map = dataList.stream().collect(Collectors.counting(x -> x));
```

## 流转换成字符串

```java
// 不使用分隔符号和前后缀, 输出结果: 144478788
List<Integer> dataList = Arrays.asList(1, 4, 4, 4, 7, 8, 7, 8, 8);
String a = dataList.stream().map(x -> "" + x).collect(Collectors.joining());
logger.debug(a);

// 仅使用分隔符, 输出结果: 1, 4, 4, 4, 7, 8, 7, 8, 8
List<Integer> dataList = Arrays.asList(1,4,4,4,7,8,7,8,8);
String a = dataList.stream().map(x -> "" + x).collect(Collectors.joining(", "));
logger.debug(a);

// 使用分隔符和前后缀, 输出结果: {1,4,4,4,7,8,7,8,8}
List<Integer> dataList = Arrays.asList(1, 4, 4, 4, 7, 8, 7, 8, 8);
String a = dataList.stream().map(x -> "" + x).collect(Collectors.joining(", ", "{",  "}"));
logger.debug(a);
```

# 组合收集器

收集器可以组合使用, 组合收集器将具备更加强大功能, 看一下下面的例子, 体验一下组合收集器的强大

## 需要实现的功能

有一个链表, 链表里面的每一个对象保存的是艺术家的名字和艺术家某一次发布的专辑的专辑名, 模拟数据如下:

```txt
艺术家：艺术家00，专辑名：专辑名00
艺术家：艺术家01，专辑名：专辑名01
艺术家：艺术家02，专辑名：专辑名02
艺术家：艺术家00，专辑名：专辑名03
艺术家：艺术家01，专辑名：专辑名04
艺术家：艺术家02，专辑名：专辑名05
艺术家：艺术家00，专辑名：专辑名06
艺术家：艺术家01，专辑名：专辑名07
艺术家：艺术家02，专辑名：专辑名08
艺术家：艺术家00，专辑名：专辑名09
艺术家：艺术家01，专辑名：专辑名10
艺术家：艺术家02，专辑名：专辑名11
艺术家：艺术家00，专辑名：专辑名12
艺术家：艺术家01，专辑名：专辑名13
艺术家：艺术家02，专辑名：专辑名14
艺术家：艺术家00，专辑名：专辑名15
艺术家：艺术家01，专辑名：专辑名16
艺术家：艺术家02，专辑名：专辑名17
艺术家：艺术家00，专辑名：专辑名18
艺术家：艺术家01，专辑名：专辑名19
艺术家：艺术家02，专辑名：专辑名20
艺术家：艺术家00，专辑名：专辑名21
艺术家：艺术家01，专辑名：专辑名22
艺术家：艺术家02，专辑名：专辑名23
艺术家：艺术家00，专辑名：专辑名24
艺术家：艺术家01，专辑名：专辑名25
艺术家：艺术家02，专辑名：专辑名26
艺术家：艺术家00，专辑名：专辑名27
艺术家：艺术家01，专辑名：专辑名28
艺术家：艺术家02，专辑名：专辑名29
```

需要对这些数据进行处理, 按照艺术家名字进行分类, 找出这个艺术家所有的专辑, 最终处理的数据使用一个 Map<String, String> 的数据结构执行, Map 的 key 为艺术家名, value 的格式为 "{专辑名01, 专辑名02, …}" 的形式, 如下所示:

```txt
key=艺术家01, value={专辑名01, 专辑名04, 专辑名07, 专辑名10, 专辑名13, 专辑名16, 专辑名19, 专辑名22, 专辑名25, 专辑名28}
```

## 不使用 Stream 时的实现方式

```java
// 生成模拟数据
Artist artist;
List<Artist> artistList = new ArrayList<>();
for (int i = 0; i < 30; i ++) {
    artist = new Artist("艺术家" + String.format("%02d", i % 3), "专辑名" + String.format("%02d", i));
    artistList.add(artist);
    logger.debug("艺术家：" + artist.getName() + "，专辑名：" + artist.getAlbums());
}

// 按照艺术家分类, nameSet 作为词典用于分类
Set<String> nameSet = new HashSet<>();
Map<String, List<String>> tempListMap = new HashMap<>();
for (Artist martist: artistList) {
    List<String> tempList;
    if (nameSet.contains(martist.getName())) {
        tempList = tempListMap.get(martist.getName());
    } else {
        nameSet.add(martist.getName());
        tempList = new ArrayList<>();
    }
    tempList.add(martist.getAlbums());
    tempListMap.put(martist.getName(), tempList);
}

for (Map.Entry<String, List<String>> entry : tempListMap.entrySet()) {
     logger.debug(entry.getKey() + " = " + entry.getValue());
}

// List 转成指定的格式
Map<String, String> finalArtistMap = new HashMap<>();
for (Map.Entry<String, List<String>> entry : tempListMap.entrySet()) {
    StringBuilder builder = new StringBuilder("{");
    List<String> albums = entry.getValue();
    for (int i = 0; i < albums.size() - 1; i++) {
        builder.append(albums.get(i));
        builder.append(", ");
    }
    if (albums.size() > 0) {
        builder.append(albums.get(albums.size() - 1));
    }
    builder.append("}");
    finalArtistMap.put(entry.getKey(), builder.toString());
}

// 输出结果
for (Map.Entry<String, String> entry : finalArtistMap.entrySet()) {
    logger.debug(entry.getKey() + " = " + entry.getValue());
}
```

输出结果如下:

```txt
艺术家01 = [专辑名01, 专辑名04, 专辑名07, 专辑名10, 专辑名13, 专辑名16, 专辑名19, 专辑名22, 专辑名25, 专辑名28]
艺术家02 = [专辑名02, 专辑名05, 专辑名08, 专辑名11, 专辑名14, 专辑名17, 专辑名20, 专辑名23, 专辑名26, 专辑名29]
艺术家00 = [专辑名00, 专辑名03, 专辑名06, 专辑名09, 专辑名12, 专辑名15, 专辑名18, 专辑名21, 专辑名24, 专辑名27]
```

## 使用 Stream 的实现方式

```java
// 生成模拟数据
Artist artist;
List<Artist> artistList = new ArrayList<>();
for (int i = 0; i < 30; i ++) {
    artist = new Artist("艺术家" + String.format("%02d", i % 3), "专辑名" + String.format("%02d", i));
    artistList.add(artist);
    logger.debug("艺术家：" + artist.getName() + "，专辑名：" + artist.getAlbums());
}

// 数据进行处理
Stream<Artist> artistStream = artistList.stream();
Map<String, String> finalArtistMap =
        artistStream.collect(Collectors.groupingBy(Artist::getName,
                Collectors.mapping(Artist::getAlbums, Collectors.joining(", ", "{", "}"))));

// 输出结果
for (Map.Entry<String, String> entry : finalArtistMap.entrySet()) {
    logger.debug(entry.getKey() + " = " + entry.getValue());
}
```

输出结果如下:

```txt
艺术家01 = {专辑名01, 专辑名04, 专辑名07, 专辑名10, 专辑名13, 专辑名16, 专辑名19, 专辑名22, 专辑名25, 专辑名28}
艺术家02 = {专辑名02, 专辑名05, 专辑名08, 专辑名11, 专辑名14, 专辑名17, 专辑名20, 专辑名23, 专辑名26, 专辑名29}
艺术家00 = {专辑名00, 专辑名03, 专辑名06, 专辑名09, 专辑名12, 专辑名15, 专辑名18, 专辑名21, 专辑名24, 专辑名27}
```

可以看到两种方式都可以得到同样的结果, 但是使用 stream 组合收集器的方式的代码更加简单和简洁