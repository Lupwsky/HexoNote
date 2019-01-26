---
title: Java-Guava工具类之字符串处理类
date: 2019-01-26 13:45:42
categories: Java
---

之前一直有听说过谷歌的 Guava, 但是在项目中也仅是非常简单使用过其中的缓存功能 , 称着这段时间项目不是很忙, 抓紧时间系统的了解下 (主要参考 [官方的文档](https://github.com/google/guava) 和 [Guava官方教程(中文版)](https://willnewii.gitbooks.io/google-guava/content/index.html)), 这里记录下自己的学习笔记

# Splitter 类

主要对比下 JDK 中 String 内建的 split() 方法对字符串处理和 Guava 提供的 Splitter 类对字符串的处理, 使用方法和差异都写在代码的注释上了

## JDK 中 String 内建的 split() 方法

```java
String value, sourceData = "A,B,C";
value = Arrays.toString(sourceData.split(","));
log.info("[splitterTest1] value = {}", value);
// 输出 [A, B, C]

// String.split() 方法默认情况下会丢弃最后末尾的分隔符
sourceData = "A,B,C,";
value = Arrays.toString(sourceData.split(","));
log.info("[splitterTest1] value = {}", value);
// 输出 [A, B, C]

// 如果不想丢弃最后末尾的分隔符, 使用重载后的方法, 借助第二个参数来实现, 使用示例如下
sourceData = "A,B,C,,,";
value = Arrays.toString(sourceData.split(",", -1));
log.info("[splitterTest1] value = {}", value);
// 输出 [A, B, C, , ,]
```

<!-- more -->

## Guava 提供的 Splitter 类

```java
// Splitter 的使用示例
// Splitter.fixedLength(int) 按固定长度拆分, 最后一段可能比给定长度短, 但不会为空字符串
String value, sourceData = "A,B,C,,";

// omitEmptyStrings() 自动忽略空字符串
value = Lists.newArrayList(Splitter.on(",").omitEmptyStrings().split(sourceData)).toString();
log.info("[splitterTest2] value = {}", value);
// 输出 [A, B, C]

// trimResults() 移除每个分隔出来的字符串的前后空格
sourceData = "A, B , C,,";
value = Lists.newArrayList(Splitter.on(",").omitEmptyStrings().split(sourceData)).toString();
log.info("[splitterTest2] value = {}", value);

value = Lists.newArrayList(Splitter.on(",").omitEmptyStrings().trimResults().split(sourceData)).toString();
log.info("[splitterTest2] value = {}", value);
// 输出 [A,  B ,  C]
// 输出 [A, B, C]

// trimResults(CharMatcher) 给给你个匹配其, 移除每个分隔出来的字符串的前后匹配的字符
sourceData = "A, B你好, 你好C,,";
value = Lists.newArrayList(Splitter.on(",")
        .trimResults(CharMatcher.anyOf("你好"))
        .omitEmptyStrings()
        .split(sourceData))
        .toString();
log.info("[splitterTest2] value = {}", value);
// 输出 [A,  B,  你好C]
// 这里的你好没有移除是因为, 分隔出来的是 " 你好C", 前面多一个空格
```

# Joiner 类

将 List = ["A", "B", "C", "D", "E"], 转换成 "A,B,C,D,E", 在 JDK 8 之前, 我们需要循环遍历进行处理

## JDK 7 的处理方式

```java
// 转换成 "A,B,C,D,E"
List<String> dataList = Lists.newArrayList("A", "B", "C", "D", "E");
String value;

// 在 String 没有提供内建的方法之前, 我们也许需要写如下一个循环来实现
if (dataList != null) {
    StringBuilder stringBuilder = new StringBuilder(dataList.get(0));
    for (int i = 1; i < dataList.size(); i++) {
        stringBuilder.append(",").append(dataList.get(i));
    }
    value = stringBuilder.toString();
} else {
    value = "";
}
log.info("[joinerTest1] value = {}", value);

// 为了避免出现异常, 需要进一步进行判空处理
// 这里为了方便使用 JDK 8 的 Optional 处理
if (dataList != null) {
    StringBuilder stringBuilder = new StringBuilder(Optional.ofNullable(dataList.get(0)).orElse(""));
    for (int i = 1; i < dataList.size(); i++) {
        stringBuilder.append(",").append(Optional.ofNullable(dataList.get(i)).orElse(""));
    }
    value = stringBuilder.toString();
} else {
    value = "";
}
log.info("[joinerTest1] value = {}", value);
// 输出 A,B,C,D,E
```

## JDK 8 的处理方式

JDK 8 中的 String 内建的 join 方法 和 Stream API 也很方便的支持这种处理, 主要使用了一个新的类 StringJoiner, 也可以用于字符串使用分隔符拼接, 并且支持前后缀

### StringJoiner 类

```java
// 单个参数分别为, 分隔符, 字符串凭借后添加的前缀, 字符串凭借后添加的后缀
StringJoiner stringJoiner = new StringJoiner(",", "[", "]");
stringJoiner.add("A");
stringJoiner.add("B");
stringJoiner.add(null);
stringJoiner.add("D");
String value = stringJoiner.toString();
log.info("[joinerTest4] value = {}", value);
// 输出 A,B,null,D
```

### String 内建的 join 方法

String 内建的 join 方法会忽略 null 和空字符串

```java
List<String> dataList = Lists.newArrayList("A", "B", "C", "D", "E");
String value = String.join(",", dataList);
log.info("[joinerTest2] value = {}", value);
// 输出 A,B,C,D,E

value = String.join(",", "A", "B", "C");
log.info("[joinerTest2] value = {}", value);
// 输出 A,B,C

// 如果是拼接的是字符串是一个 null, 会将 "null", 字符串拼接
// 以下例子输出的 value 值为 "A,B,C,null,null", 对于 null 需要进行特殊的额处理
dataList = Lists.newArrayList("A", "B", "C", null, null);
value = String.join(",", dataList);
log.info("[joinerTest2] value = {}", value);
// 输出 A,B,C,null,null

value = String.join(",", "A", "B", "C", null, null);
log.info("[joinerTest2] value = {}", value);
// 输出 A,B,C,null,null
```

### Stream API 中使用

Collectors.joining() 方法会忽略 null 和空字符串

```java
List<String> dataList = Lists.newArrayList("A", "B", null, null, "E");
value = dataList.stream().filter(Objects::nonNull).collect(Collectors.joining(",", "[", "]"));
log.info("[joinerTest4] value = {}", value);
// 输出 A,B,D
```

## Guava 提供的 Joiner 类

### skipNulls() 跳过 null

```java
List<String> dataList = Lists.newArrayList("A", "B", null, null, "E");
// skipNulls() 方法可以跳过所有 null 的数据
String value = Joiner.on(",").skipNulls().join(dataList);
log.info("[joinerTest3] value = {}", value);
// 输出 A,B,E
```

### useForNull() 方法替代 null

```java
// useForNull() 方法替代 null
value = Joiner.on(",").useForNull("NONE").join(dataList);
log.info("[joinerTest3] value = {}", value);
// 输出 A,B,NONE,NONE,E
```

### 对 Map 的键值组合后进行拼接

```java
// Joiner 类还可以对 Map 里面的键值组合后进行拼接
HashMap<String, String> hashMap = Maps.newHashMap();
hashMap.put("K1", "1");
hashMap.put("K2", "2");
hashMap.put("K3", "3");
value = Joiner.on(",").withKeyValueSeparator("/").join(hashMap);
log.info("[joinerTest3] value = {}", value);
// 输出 K1/1,K2/2,K3/3
```