---
title: Java-Java8函数式接口
date: 2018-10-18 23:19:22
categories: Java
---

在了解 Java 8 的函数表达式之前, 先了解下 Lambda 表达式, Lambda 表达式是一种可传递匿名表达式的一种方式, 可以作为参数传递或者存储在变量中, 使用 Lambda 可以让`行为参数化` (关于行为参数见 `Java-行为参数化` 笔记) 更加的灵活

# Lamdbda 的语法

## (parameters) -> expression

这种语法函数体是一个表达式, 表达式后面不需要分号, 不需要带大括号, 例 `(x) -> "TEST"`

## (parameters) -> {statements;}

这种语法的函数体是语句, 语句后面需要带分号, 且需要带大括号, 例 `(x) -> {return "TEST";}`

# 可以使用 Lambda 表达式的地方

在`函数式接口上`可以使用 Lambda 表达式

<!-- more -->

# 函数式接口

函数式接口是之定义了一个抽象方法的接口, 在 Java 8 中的接口开始运行拥有多个默认方法, 即使有多个默认方法, 只要只有一个抽象方法任然还是一个函数式接口

# 函数式接口和 Lambda 表达式

Lambda 表达式是函数式接口的一个具体实例 (使用内部匿名类也可以完成 Lambda 表达式一样的功能, 但是比较笨拙)

# Java 8 中的新增的函数式接口

Java 8 的 java.util.function 包中引入了一些新的函数式接口, 如 Predicate, Consumer 和 Function, 现在分别介绍这几个接口

## Predicate

Predicate 接口里面有一个返回 boolean 类型的 test 抽象方法和其他几个默认实现方法, 当调用 test 方法时会执行传递进来的 Lambad 表达式的方法体里面的代码并返回一个 boolean 对象, 例如实现一个类似 Stream API 中的 filter 方法这种需求可以使用这个接口, Predicate 接口的源码如下:

```java
@FunctionalInterface
public interface Predicate<T> {
    boolean test(T t);

    default Predicate<T> and(Predicate<? super T> other) {
        Objects.requireNonNull(other);
        return (t) -> test(t) && other.test(t);
    }

    default Predicate<T> negate() {
        return (t) -> !test(t);
    }

    default Predicate<T> or(Predicate<? super T> other) {
        Objects.requireNonNull(other);
        return (t) -> test(t) || other.test(t);
    }
}
```

Predicate 可用于需要返回一个 boolean 类型的 Lambda 表达式, 下面实现一个类似 Stream API 中的 filter 的接口, 对 List 进行过滤, 使用示例如下:

```java
public static void main(String[] args) {
    List<DataNode> dataNodeList = new ArrayList<>();
    dataNodeList.add(DataNode.builder().name("A").soc(1).build());
    dataNodeList.add(DataNode.builder().name("B").soc(1).build());
    dataNodeList.add(DataNode.builder().name("C").soc(1).build());
    dataNodeList.add(DataNode.builder().name("D").soc(1).build());
    dataNodeList.add(DataNode.builder().name("E").soc(2).build());
    dataNodeList.add(DataNode.builder().name("F").soc(2).build());
    dataNodeList.add(DataNode.builder().name("G").soc(2).build());
    dataNodeList.add(DataNode.builder().name("H").soc(2).build());
    dataNodeList.add(DataNode.builder().name("I").soc(3).build());
    dataNodeList.add(DataNode.builder().name("J").soc(3).build());
    dataNodeList.add(DataNode.builder().name("K").soc(3).build());
    dataNodeList.add(DataNode.builder().name("L").soc(3).build());

    // 过滤掉 sco 值大于 2 的元素
    List<DataNode> filterDataList = filter(dataNodeList, (dataNode -> dataNode.getSoc() > 2));
    log.info(filterDataList.toString());
}

private static List<DataNode> filter(List<DataNode> sourceDataList, Predicate<DataNode> predicate) {
    List<DataNode> returnDataList = new ArrayList<>();
    sourceDataList.forEach(dataNode -> {
        if (predicate.test(dataNode)) {
            returnDataList.add(dataNode);
        }
    });
    return returnDataList;
}
```

输出结果如下:

```txt
[DataNode(name=I, soc=3), DataNode(name=J, soc=3), DataNode(name=K, soc=3), DataNode(name=L, soc=3)]
```

Predicate 接口还默认实现了 and, or 和 negate 方法, 这些方法可以和 test 组合使用, 例如将上面的 filter 方法改成既要满足 Lambda 表达式的刷选条件, 又要满足元素的 name 是 L 的元素, 代码如下:

```java
private static List<DataNode> filter(List<DataNode> sourceDataList, Predicate<DataNode> predicate) {
    List<DataNode> returnDataList = new ArrayList<>();
    sourceDataList.forEach(dataNode -> {
        if (predicate.and(dataNodec -> {
            log.info("AND    = " + dataNodec.toString());
            return dataNodec.getName().equals("L");
        }).test(dataNode)) {
            log.info("TEST   = " + dataNode.toString());
            returnDataList.add(dataNode);
        }
    });
    return returnDataList;
}

List<DataNode> filterDataList = filter(dataNodeList, (dataNode -> dataNode.getSoc() > 2));
log.info("RESULT = " + filterDataList.toString());
```

输出结果如下:

```txt
[main] - AND    = DataNode(name=I, soc=3)
[main] - AND    = DataNode(name=J, soc=3)
[main] - AND    = DataNode(name=K, soc=3)
[main] - AND    = DataNode(name=L, soc=3)
[main] - TEST   = DataNode(name=L, soc=3)
[main] - RESULT = [DataNode(name=L, soc=3)]
```

从输出结果可以看到最终的结果集都是 name = "L" 并且 sco > 3 的元素, 也可以了解到 and, or 和 negate 方法都是在 test 方法过滤元素之前执行的

## Comsumer

Consumer 接口里面定义了一个 accept 的抽象方法, 接收一个泛型参数, 返回一个 void 类型, 另外一个默认的实现方法 addThen 可以在 accept 方法执行后再次对数据进行一次操作, 如果仅需要对某个数据进行操作, 可以使用次接口, 类似于 Stream API 中的 forEach 方法, Consumer 接口的源码如下:

```java
@FunctionalInterface
public interface Consumer<T> {
    void accept(T t);
    
    default Consumer<T> andThen(Consumer<? super T> after) {
        Objects.requireNonNull(after);
        return (T t) -> { accept(t); after.accept(t); };
    }
}
```

实现一个类似 Stream API 中的 forEach 方法的效果, 使用示例如下:

```java
public static void main(String[] args) {
    List<DataNode> dataNodeList = new ArrayList<>();
    dataNodeList.add(DataNode.builder().name("A").soc(1).build());
    dataNodeList.add(DataNode.builder().name("B").soc(2).build());
    dataNodeList.add(DataNode.builder().name("C").soc(3).build());
    dataNodeList.add(DataNode.builder().name("D").soc(4).build());

    Consumer<DataNode> consumer = dataNode -> {
        int i = dataNode.getSoc();
        i += 1;
        dataNode.setSoc(i);
        log.info("ACCEPT  = " + dataNode.toString());
    };
    forEach(dataNodeList, consumer);
    log.info("RESULT  = " + dataNodeList.toString());
}

// addThen 方法可以在每次执行了 accept 方法后执行一次其他的操作
private static void forEach(List<DataNode> dataNodeList, Consumer<DataNode> consumer) {
    for (DataNode dataNode : dataNodeList) {
        consumer.andThen(node -> {
            node.setName(node.getName() + "1");
            log.info("ADDTHEN = " + node.toString());
        }).accept(dataNode);
    }
}
```

输出结果如下:

```txt
[main] - ACCEPT  = DataNode(name=A, soc=2)
[main] - ADDTHEN = DataNode(name=A1, soc=2)

[main] - ACCEPT  = DataNode(name=B, soc=3)
[main] - ADDTHEN = DataNode(name=B1, soc=3)

[main] - ACCEPT  = DataNode(name=C, soc=4)
[main] - ADDTHEN = DataNode(name=C1, soc=4)

[main] - ACCEPT  = DataNode(name=D, soc=5)
[main] - ADDTHEN = DataNode(name=D1, soc=5)
[main] - RESULT  = [DataNode(name=A1, soc=2), DataNode(name=B1, soc=3), DataNode(name=C1, soc=4), DataNode(name=D1, soc=5)]
```

## Function

Function 接口定义了一个 apply 的接口, 接收一个泛型类型 T, 并返回一个泛型 R, 其中还有两个默认实现的方法, compose 和 andThen, 这两个方法分别在 apply 方法执行前调用和执行后再次对数据进行一次映射, 如果需要将某个类型映射成其他的类型, 可以使用该接口, 和 Stream API 中的 map 方法类似, Function 源码如下:

```java
@FunctionalInterface
public interface Function<T, R> {
    R apply(T t);

    default <V> Function<V, R> compose(Function<? super V, ? extends T> before) {
        Objects.requireNonNull(before);
        return (V v) -> apply(before.apply(v));
    }

    default <V> Function<T, V> andThen(Function<? super R, ? extends V> after) {
        Objects.requireNonNull(after);
        return (T t) -> after.apply(apply(t));
    }
}
```

实现一个简单的 map 方法, 将定义好的 DataNode 的列表映射成 String 类型的列表, 使用示例如下:

```java
public static void main(String[] args) {
    List<DataNode> dataNodeList = new ArrayList<>();
    dataNodeList.add(DataNode.builder().name("A").soc(1).build());
    dataNodeList.add(DataNode.builder().name("B").soc(2).build());
    dataNodeList.add(DataNode.builder().name("C").soc(3).build());
    dataNodeList.add(DataNode.builder().name("D").soc(4).build());

    List<String> list = map(dataNodeList, DataNode::getName);
    log.info(list.toString());
}

private static List<String> map(List<DataNode> dataNodeList, Function<DataNode, String> function) {
    List<String> returnDataList = new ArrayList<>();
    for (DataNode dataNode : dataNodeList) {
        returnDataList.add(function.apply(dataNode));
    }
    return returnDataList;
}
```

输出结果如下:

```txt
[main] - [A, B, C, D]
```

## Supplier

Supplier 接口只有一个抽象的 get 方法, 不接受任何参数, 返回一个泛型 T 参数, 比较简单就不举例了, 源码如下:

```java
@FunctionalInterface
public interface Supplier<T> {
    T get();
}
```

## 其他的接口

上面说到的 Predicate, Comsumer 和 Function 是 java.util.function 提供的接口, 除了这些接口外, 还提供了一些其他的接口, 但都是和 Predicate, Comsumer 和 Function 相关的接口, 主要分成以下两类:

* BiXXX 类型的接口, 如 BiFunction 接口, 这类接口和 Predicate, Comsumer 和 Function 接口功能一样, 不同的是这类接口有两个入参
* ObjXXXYYY 类型接口, 如 ObjIntFunction 接口, 这类接口有两个入参, 只是第一个参数是对象类型, 第二个参数是基础数据类型 (int, long, double)
* XXXToYYYFunction, 如 LongToIntFunction 接口, 基础类型 long 映射 int 类型使用
* 最后还有一类是为了不让处理数据时对基础类型进行自动的拆箱和装箱操作影响性能而创建的诸如 ToIntFunction 等接口

# 参考资料

* Java 8 实战 (书籍)