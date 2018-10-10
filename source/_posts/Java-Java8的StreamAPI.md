---
title: Java-Java8的StreamAPI
date: 2018-10-10 23:29:47
categories: Java
---

Java 8 中引入了全新的 Stream API, 这里的 Stream 和 I/O 中的 Stream 不同, 他更像是具有 Iterable 的集合, 但是行为和集合类又不相同, Stream API 引入的目的在于弥补 Java 函数式编程的缺陷。对于很多支持函数式编程的语言, map()、reduce() 基本上都内置到语言的标准库中了, 不过, Java 8 的 Stream API 总体来讲仍然是非常完善和强大，足以用很少的代码完成许多复杂的功能

# 创建 Stream 对象

创建的方法有很多, 最简单的方法就是将 Collection 转变成一个 Stream, 看下面一个例子:

```java
public static void main(String[] args) {
    List<Integer> numbers = Arrays.asList(1, 2, 3, 4, 5, 6, 7, 8, 9, 10);
    Stream<Integer> stream = numbers.stream();
    stream.filter((x) -> {
        return x % 2 == 0;
    }).map((x) -> {
        return x * x;
    }).forEach(System.out::println);
}
```

<!-- more -->

集合类新增的 stream() 方法用于把一个集合变成 Stream, 然后, 通过 filter()、map() 等实现 Stream 的变换, Stream 还有一个 forEach() 来完成每个元素的迭代

# 常用方法

## filter 方法

filter 方法主要用于过滤出符合条件的 Stream 中的元素，示例代码如下:

```java
// 示例 1
List<String> stringList = Arrays.asList("nanjing", "tianjin", "shangrao", "shanghai", "nanchang");
stringList.stream().filter((x) -> x.leght > 7).forEach(System.out::println);

// 示例 2
List<String> dataList = Arrays.asList("1", "2", "3", "4", "5", "6", "7");
Stream<String> dataStream = dataList.stream();
dataStream.forEach(x -> logger.debug("dataStream：" + x));

// 示例 3
Stream<String> filterStream = dataList.stream().filter(x -> x.equals("2") || x.equals("3"));
filterStream.forEach(x -> logger.debug("filter：" + x));
```

## map 方法

用于对Stream中的元素转换成其他的值, 示例代码如下:

```java
Stream.of("Bejing", "Shanghai", "Nanjing").map(String::toUpperCase).forEach(System.out::println);
List<Map<String, Object>> mapList = dataList.stream()
    .map(x -> {
        HashMap<String, Object> map = new HashMap<>();
        map.put("index", x);
        return map;
    }).collect(Collectors.toList());
```

## mapToInt 方法

mapToInt 用于将 Stream 中的对象转换成 int 类型, 其他类似的还有 mapToLong, mapToDouble等等, 示例代码如下:

```java
Stream.of("1", "2", "3").mapToInt(x -> Integer.valueOf(x)).forEach(System.out::println);
```

## flatMap 方法

用于将 Stream 中的每个元素都转换成 Stream 类型, 然后将每个 Stream 的元素提取出来, 聚合成一个最外围的 Stream, 示例代码如下:

```java
Stream.of(Arrays.asList(1, 2), Arrays.asList(3, 4))
       .flatMap(List::stream)
       .forEach(System.out::println);
```

## distinct 方法

将 Stream 中相等的元素去除, 通过 equals 方法来判断元素是否相等, 示例代码如下:

```java
// 结果输出为 1, 2, 3, 4
Stream.of(1, 2, 3, 4, 2, 3, 1).distinct().forEach(System.out::println);  
```

## sorted 方法

sorted 方法是将 Stream 中的元素进行排序, 它内部的元素必须实现 Comparable, sorted 方法可以接收一个 Comparator, 示例代码如下:

```java
 Stream.of("nanjing", "beijing", "nantong", "shangrao").sorted().forEach(System.out::println);
```

## forEach 方法

forEach 方法主要是对 Stream 中每个元素进行操作, 示例如下:

```java
Stream.of(1, 2, 3).forEach(System.out::println);
```

## reduce 方法

reduce 方法是一个聚合方法, 是将 Stream 中的元素进行某种操作, 得到一个元素, 示例代码如下:

```java
// 将 Stream 的每个元素相加，并加上初始值 0，reduce 还有 2 个重载的方法
System.out.println(Stream.of(1, 2, 3).reduce(0, (acc, element) -> acc + element));
```

## collect 方法

collect 方法是将 Stream 转换成一个 List, 是一个及早求值, 示例代码如下:

```java
// 可以传入Collectors.toSet()等其他值
System.out.println(Stream.of("abc", "def", "efg").collect(Collectors.toList())); 
```

## min 和 max 方法

min, max方法是获取 Stream 中最小或最大的元素的值, 需要传递一个 Comparator 比较器参数, 是聚合方法, 示例代码如下:

```java
List<String> cities = Arrays.asList("nanjing", "beijing", "nantong", "haimen", "shangrao");
System.out.println(cities.stream().min(Comparator.comparing(String::length)).get());
```

## count 方法

count 方法是获取 Stream 中元素的数量, 是聚合方法, 示例代码如下:

```java
System.out.println(Stream.of(1, 2, 3).count());
```

## anyMatch 方法

anyMatch 方法用来判断 Stream 中是否存在指定条件的元素, 示例代码如下:

```java
// 输出 true
System.out.println(Stream.of(1, 2, 3).anyMatch(data -> data > 2));

// 输出 false
System.out.println(Stream.of(1, 2, 3).anyMatch(data -> data > 3));  
```

## allMatch 方法

allMatch 方法用来判断 Stream 中的元素是否都满足指定的条件, 示例代码如下:

```java
// 输出 false
System.out.println(Stream.of(1, 2, 3).allMatch(data -> data > 2));

// 输出 true  
System.out.println(Stream.of(1, 2, 3).allMatch(data -> data > 0));  
```

## noneMatch 方法

noneMatch 方法用来判断 Stream 中是否所有元素都不满足指定条件, 示例代码如下:

```java
// 输出 false
System.out.println(Stream.of(1, 2, 3).noneMatch(data -> data > 2));

// 输出 true
System.out.println(Stream.of(1, 2, 3).noneMatch(data -> data > 3));  
```

## findFirst 方法

findFirst 方法返回 Stream 中的第一个元素, 返回的类型为 Optional, 示例代码如下:

```java
System.out.println(Stream.of(1, 2, 3).findFirst().get());
```

# 及早求值和惰性求值

通常Java中调用一个方法, 计算机会随即执行操作, 比如 System.out.println, 调用后会立即在控制台输出一条信息, 但是在 Strema 里面的方法确不相同, 他们虽然是普通的方法, 但是返回的 Stream 不是一个集合 (Stream 虽然可以理解为一个高级集合, 但是他本身并不是一个集合), 只能通过 collect(toList()) 转化成一个集合, 例如: players.filter(string -> string.toUpCase()), 他返回了一个了一个Stream对象，如果在将代码改动如下:

```java
players.filter(string -> {
    System.out.println(string.toUpCase());
    return true;
});
```

可以看到, 执行代码后在控制台没有任何的输出, 像这种只描述 Stream 确不产生集合的方法称之为惰性求值方法, 而像 count 这样最终依靠 Stream 产生其他值或者集合的方法称为及早求值方法, 如果将上面代码改成以下代码, 将会在控制台输出相关信息:

```java
players.filter(string -> {
    System.out.println(string.toUpCase());
    return true;
}).count();
```

<div class="note default"><p> 判断一个操作是否是惰性求值或者及早求值的方法: 如果方法的返回值是一个 Stream 对象, 那么就是一个惰性求值方法, 如果返回的是一个值或者集合, 那么就是及早求值方法 </p></div>