---
title: Java-String、StringBuffer和StringBuilder
date: 2018-05-11 18:17:58
categories: Java
---

# String

- 一个字符串是一个匿名的 String 类
- 可以直接把字符串赋值给一个 String 类的对象，也可以通过 new 的方法，前者可以自动入池，也不产生垃圾空间
- String 类的内容不可改变
- String 不适合频繁的修改
- String 拼接字符串的效率也很低

# String 和 StringBuffer 互相转换

## String 转 StringBuffer

利用 StringBuffer 的构造器

```java
StringBuffer stringBuffer = new StringBuffer(String str);
```

<!-- more -->

利用 StringBuffer 的 append 方法

```java
String str = "A";
StringBuffer stringBuffer = new StringBuffer();
stringBuffer.append(str);
```

## StringBuffer 转 String

利用 StringBuffer 的 toString 方法

```java
String string = stringBuffer .toString()
```

利用 "+" 符号

```java
String string = stringBuffer + ""
```

# StringBuffer 和 StringBuilder 区别

StringBuffer 是在 JDK1.0 提供的，在 JDK1.5 以后提供了一个 StringBuilder 类，StringBuffer和StringBuilder类的方法几乎是一样的，不同的是StringBuffer是线程安全的，性能相较于StringBuilder类要低，而StringBuilder类属于非线程安全的类，性能更高。

# 三者的异同

(1) 都是继承于 CharSequence 类
(2) StringBuffer 属于线程安全类，StringBuilder 类属于非线程安全类
(3) String 类的内容是不可变的，StringBuffer 和 StringBuilder 类是可以变的
(4) String 不适合频繁的修改，StringBuffer 和 StringBuilder 类适合频繁修改
(5) String 拼接字符串的效率也很低，每一个拼接符号 "+"，都需要 new 一个对象出来，而 StringBuffer 和 StringBuilder 不需要