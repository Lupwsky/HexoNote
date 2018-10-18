---
title: Java-Java8之StreamAPI中的faltMap方法
date: 2018-10-18 23:21:49
tags:
---

# flatMap 方法

flatMap 方法可以将多个 Stream 连接成一个 Stream, 可以简单理解为将 List<List<T>> 合并或转换成 List<T> 类型

使用示例如下:

```java
// DataNode 只有一个 String 类型的成员变量 name 和 int 类型的成员变量 soc
List<DataNode> dataNodeList1 = new ArrayList<>();
dataNodeList1.add(DataNode.builder().name("A").soc(1).build());
dataNodeList1.add(DataNode.builder().name("B").soc(1).build());
dataNodeList1.add(DataNode.builder().name("C").soc(1).build());
dataNodeList1.add(DataNode.builder().name("D").soc(1).build());

List<DataNode> dataNodeList2 = new ArrayList<>();
dataNodeList2.add(DataNode.builder().name("E").soc(1).build());
dataNodeList2.add(DataNode.builder().name("F").soc(1).build());
dataNodeList2.add(DataNode.builder().name("G").soc(1).build());
dataNodeList2.add(DataNode.builder().name("H").soc(1).build());

List<DataNode> dataNodeList3 = new ArrayList<>();
dataNodeList3.add(DataNode.builder().name("I").soc(1).build());
dataNodeList3.add(DataNode.builder().name("J").soc(1).build());
dataNodeList3.add(DataNode.builder().name("K").soc(1).build());
dataNodeList3.add(DataNode.builder().name("L").soc(1).build());

List<List<DataNode>> dataList = new ArrayList<>();
dataList.add(dataNodeList1);
dataList.add(dataNodeList2);
dataList.add(dataNodeList3);

List<DataNode> newDataList = dataList.stream().flatMap(Collection::stream).collect(Collectors.toList());
log.info(newDataList.toString());
```

输出结果如下:
```txt
[
    DataNode(name=A, soc=1),
    DataNode(name=B, soc=1), 
    DataNode(name=C, soc=1), 
    DataNode(name=D, soc=1), 
    DataNode(name=E, soc=1), 
    DataNode(name=F, soc=1), 
    DataNode(name=G, soc=1), 
    DataNode(name=H, soc=1), 
    DataNode(name=I, soc=1), 
    DataNode(name=J, soc=1),
    DataNode(name=K, soc=1), 
    DataNode(name=L, soc=1)
]
```

# 参考资料

* Java 8 实战 (书籍)
