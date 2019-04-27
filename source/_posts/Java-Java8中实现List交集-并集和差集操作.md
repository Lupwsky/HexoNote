---
title: 'Java-Java8中实现List交集,并集和差集操作'
date: 2019-04-01 21:44:55
categories: Java
---

# Java8 之前实现 Lsit 交集, 并集和差集操作

在 Java 8 之前求 List 的交集, 并集和差集操作, 如下:

```java
// 求交集
List<String> aList = Lists.newArrayList("A", "B", "C");
List<String> bList = Lists.newArrayList("B", "C", "D");
aList.retainAll(bList);

// 求并集, 有重复值
List<String> aList = Lists.newArrayList("A", "B", "C");
List<String> bList = Lists.newArrayList("B", "C", "D");
aList.addAll(bList);

// 求并集, 无重复值
List<String> aList = Lists.newArrayList("A", "B", "C");
List<String> bList = Lists.newArrayList("B", "C", "D");
bList.removeAll(aList);
aList.addAll(bList);

// 求差集
List<String> aList = Lists.newArrayList("A", "B", "C");
List<String> bList = Lists.newArrayList("B", "C", "D");
aList.removeAll(bList);
```

<!-- more -->

# Java8 中实现 Lsit 交集, 并集和差集操作

上面在 Java 8 之前求 List 的交集, 并集和差集有一个缺点就是破坏了原 List 的完整性, 里面的元素被添加或者被删除, 如果在业务中在操作后还需要使用原 List, 需要在操作前创建一个新的 List, 并做一次深度拷贝, 将原 List 的值拷贝到新的 List 中, 后续使用的时候使用新的 List, 在 Java 8 中可以使用 Stream API 中的 filter 方法来实现这些操作, 可以保持原 List 的完整性, 在我的项目中的实现代码如下:

```java
// 求交集
List<String> oldFuncCodeList = request.getOldFuncCodeList();
List<String> newFuncCodeList = request.getNewFuncCodeList();
List<String> dataList = newFuncCodeList.stream().filter(funcCode -> oldFuncCodeList.contains(funcCode)).collect(Collectors.toList());

// 求并集, 有重复值
List<String> oldFuncCodeList = request.getOldFuncCodeList();
List<String> newFuncCodeList = request.getNewFuncCodeList();
List<String> dataList = Stream.concat(newFuncCodeList.stream(), oldFuncCodeList.stream()).collect(Collectors.toList());

// 求并集, 无重复值
List<String> oldFuncCodeList = request.getOldFuncCodeList();
List<String> newFuncCodeList = request.getNewFuncCodeList();
List<String> dataList = Stream.concat(newFuncCodeList.stream(), oldFuncCodeList.stream()).distinct().collect(Collectors.toList());

// 求差集
List<String> oldFuncCodeList = request.getOldFuncCodeList();
List<String> newFuncCodeList = request.getNewFuncCodeList();
List<String> dataList = newFuncCodeList.stream().filter(funcCode -> !oldFuncCodeList.contains(funcCode)).collect(Collectors.toList());
```

`注: 上面的求并集并无重复值的 distinct() 在自定义对象里面无法使用, 直接使用只有简单的 String, Integer 类型可以生效, 对于自定的对象, 需要对自定义对象的 equals() 和 hashcode() 方法`
