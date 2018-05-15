---
title: Java-List删除数据
date: 2018-05-14 14:17:58
categories: Java
---

有时候需要循环遍历删除 List 里面符合某一种条件的数据，我在工作的时候也遇到过需要删除 List 中的数据的情况，确遇见了 ConcurrentModificationException 异常，这篇笔记主要记录 List 如何删除数据。

# foreach 方式删除数据出现 ConcurrentModificationException 异常

首先我使用 foreach 方法循环的方法删除数据出现 ConcurrentModificationException 异常，代码如下。

```java
public static void main(String[] args) {
    List<String> list = new ArrayList<>();
    list.add("1");
    list.add("1");
    list.add("2");
    list.add("3");
    list.add("4");
    list.add("5");

    for (String value : list) {
        if (value.equals("1"))
            list.remove(value);
    }
}
```

<!-- more -->

执行这段代码，出现了 ConcurrentModificationException 异常，因为 ArrayList 并不是线程安全的，和 HashMap 等集合一样，他使用了 Fast-Fail 机制，在 ArrayList 里面有一个 modCount 值，每次新增、删除和修改元素都会让这个值自增，foreach (增强型 for 语句)本质上也是一个迭代器(普通的 for 语句不是一个迭代器)，在迭代器初始化的时候，迭代器会获取到这个值，在每次循环开始前都会比对这个值，如果不相同就会出现 ConcurrentModificationException 异常。

# for 循环删除数据出现 IndexOutOfBoundsException 异常

既然使用 foreach 会出现 ConcurrentModificationException 异常，那就不使用 foreach，使用 for 语句循环吧，所以将代码改成如下。

```java
int size = list.size();
for (int i = 0; i < size; i++) {
    if (list.get(i).equals("1"))
        list.remove(list.get(i));
}
```

执行这段代码，但是还是出现了 IndexOutOfBoundsException 异常，因为删除后 list 的 size 和循环开始时获取的 size 相比减小了，既然这样，每次循环的时候获取size不就可以了，代码改成如下。

```java
for (int i = 0; i < list.size(); i++) {
    if (list.get(i).equals("1"))
        list.remove(list.get(i));
}
```

执行这段代码，虽然没有报错，但是这样做可能删除的数据并不是自己想要删除的，为了能正确的删除，每次删除一个就 return，代码更改如下。

```java
for (int i = 0; i < list.size(); i++) {
    if (list.get(i).equals("1")) {
        list.remove(list.get(i));
        return;
    }
}
```

使用迭代器和 for 语句循环删除，每次找到符合条件的删除当前元素，马上退出循环，这样做也可以删除元素，如果只是删除一个元素倒好，如果删除多个元素，这种方法处理起来就很复杂了，且效率也很低。

# 最终的解决方法

## 使用迭代器的 remove 方法

```java
Iterator<String> iterator = list.iterator();
while (iterator.hasNext()) {
    if (iterator.next().equals("1")) {
        iterator.remove();
    }
}
```

## Java8 新增的 removeIf 方法

在Java8后支持Lambda表达式后，List新增了removeIf的Lambda方法，上面删除的语句可以更加简洁，代码如下。

```java
list.removeIf(s -> s.equals("1"));
```