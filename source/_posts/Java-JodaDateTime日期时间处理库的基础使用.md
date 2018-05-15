---
title: Java-JodaDateTime日期时间处理库的基础使用
date: 2018-05-14 14:39:49
categories: Java
---

Joda-Time，一个面向 Java 平台的易于使用的开源时间日期库，Joda-Time 可以用来替换 JDK 的日期处理类，并且比 JDK中的 Date 类更加优秀。在这之前我一直也是在使用这个库，但是 Java8 中也提供了新的时间处理类，也很方便，之后的笔记在记录 Java8 中新的时间处理类的使用方法，这一篇笔记就看一看 Joda-Time 库如何使用。

# 官方地址

GitHub：`https://github.com/JodaOrg/joda-time`
官网：`http://www.joda.org/joda-time/`

Gradle 方式引用：

```ini
compile 'joda-time:joda-time:2.9.6'
```

Maven 方式引用：

```xml
<dependency>
    <groupId>joda-time</groupId>
    <artifactId>joda-time</artifactId>
    <version>2.9.7</version>
</dependency>
```

<!-- more -->

# 使用方法

## 初始化

DateTime 类作为最重要和最核心的一个类，Joda-Time 为它提供了丰富的初始化方式。

```java
// 以当前系统的毫秒级时间构建实例
DateTime dateTime;dateTime = new DateTime();

// 以毫秒级时间参数构建实例
DateTime dateTime = new DateTime(1481006233254L);

// 以String为参数构造实例
DateTime dateTime = new DateTime("2016-11-22");

// 以 年.月.日.时.分.秒 构造实例
DateTime dateTime = new DateTime(2016,12,1,11,22,59);

// 以 年.月.日.时.分.秒.毫秒 构造实例
DateTime dateTime = new DateTime(2016,12,1,11,22,59,114);

// 以 JDK 中 Date 为参数构造实例
DateTime dateTime = new DateTime(new Date());

// 以 JDK中Calendar为参数构造实例
DateTime dateTime = new DateTime(Calendar.getInstance());

// 以 DateTime本身为参数构造实例
DateTime dateTime = new DateTime(new DateTime());
```

以上的每种实例化方式都可以在最后加上DateTimeZone参数来指定时区或者加上Chronology参数指定年表，两个参数都不传或者传null，将会使用默认值。除此之外，还能够自己定义格式转换。

```java
// 通过 DateTime 的 parse 方法解析时间
DateTimeFormatter format = DateTimeFormat.forPattern("yyyy-MM-dd HH:mm:ss"); 
DateTime dateTime = DateTime.parse("2017-12-01 14:22:45", format);
```

## 获取 Unix 时间戳

当时在使用这个库的时候，没有找到直接获取 Unix 时间戳的方法，包括初始化也没有使用时间戳初始化的方法，所以我再使用的时候都是使用 Unix 时间戳进行乘以 1000 后作为参数去创建 DateTime 实例，获取 Unix 时间戳也是获取毫秒数后除以 1000 来获取的。

```java
dateTime = new DateTime(unixStamptime *  1000);
int times = dateTime.getMi.. / 1000;
```

## 格式化输出

标准的写法是通过 DateTimeFormatter 来格式化，代码如下：

```java
DateTime dateTime = new DateTime();
// 使用DateTimeFormatter 格式化日期
DateTimeFormatter fmt = DateTimeFormat.forPattern("yyyy-MM-dd");
// DateTimeFormatter 作为参数传到 toString 方法中
String time = dateTime.toString(fmt);
```

更加简单的方式，DateTime 的 toString 方法有一个 String 类型参数的重载 toString(formatter) 可以方便格式化输出，相关示例代码如下：

```java
DateTime dateTime = new DateTime();
String time = dateTime.toString("yyyy-MM-dd");

// 我们可以更方便输出更多格式
dateTime.toString("MM/dd/yyyy hh:mm:ss.SSSa");
dateTime.toString("dd-MM-yyyy HH:mm:ss");
dateTime.toString("EEEE dd MMMM, yyyy HH:mm:ssa");
dateTime.toString("MM/dd/yyyy HH:mm ZZZZ");
dateTime.toString("MM/dd/yyyy HH:mm ZZZZ");
dateTime.toString("MM/dd/yyyy HH:mm Z");
```

## 日期的比较

比较的方法有 isEqual，isBefore 和 isAfter，每个方法对应三种重载，同时还有和当前时间比较的方法，示例代码如下：

```java
DateTime dateTime = new DateTime("2014-11-22");
DateTime dateTime2 = new DateTime("2014-11-22");
DateTime dateTime3 = new DateTime("2015-12-25");

// true
dateTime.isEqual(dateTime2);

// true
dateTime.isBefore(dateTime3);
```

## 日期的操作

### 设置日期

设置日期的方法都是以 with 开头，示例代码如下：

```java
// 初始化一个 DateTime
DateTime dateTime = new DateTime("2016-3-31");
dateTime.toString("yyyy-MM-dd hh:mm:ss");//2016-03-31 12:00:00

// 将年设置为 2016
// 前面说过，所有更改时间的操作都会返回新的 DateTime
// 旧的 DateTime 日期值不会改变
DateTime dateTime1 = dateTime.withYear(2016);

// 2014-03-31 12:00:00
dateTime1.toString("yyyy-MM-dd hh:mm:ss");
```

### 增加日期

增加日期的方法都是以plus开头，示例代码如下：

```java
DateTime dateTime = new DateTime();
dateTime.toString("yyyy-MM-dd hh:mm:ss");

// 当前时间基础上加上两天
DateTime dateTime2 = dateTime.plusDays(2);
dateTime2.toString("yyyy-MM-dd hh:mm:ss");
```

### 减少日期

减少日期的方法都是以minus开头，示例代码如下：

```java
DateTime dateTime = new DateTime();
dateTime.toString("yyyy-MM-dd hh:mm:ss");

// 减少两天
DateTime dateTime2 = dateTime.minusDays(2);
dateTime.toString("yyyy-MM-dd hh:mm:ss");

// 减少五个月，并返回一个新的DateTime
DateTime dateTime2 = dateTime.minusMonths(5);
dateTime2.toString("yyyy-MM-dd hh:mm:ss");
```

### 获取常用属性

通过get开头的方法，可以获取一些常用的属性，示例代码如下：

```java
DateTime dateTime = new DateTime("2016-3-31");
int dayOfWeek = dateTime.getDayOfWeek();
logger.i("tag", "3月31是星期：" + dayOfWeek);
```

Property 是 DateTime 中的属性，使用 Property 可以做一些 get 方法无法进行的操作：

```java
DateTime dateTime = new DateTime("2018-3-31");

// Thu
dateTime.dayOfWeek().getAsShortText(Locale.ENGLISH);

// 周四
dateTime.dayOfWeek().getAsShortText(Locale.CHINA);

// 목요일 (韩语:星期四)
dateTime.dayOfWeek().getAsText(Locale.KOREAN);

 //星期四
dateTime.dayOfWeek().getAsText(Locale.CHINA);

// getAsShortText：获取缩略的文本
// getAsText： 获取完整的文本
```