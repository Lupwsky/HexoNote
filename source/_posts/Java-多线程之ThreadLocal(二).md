---
title: Java-多线程之ThreadLocal(二)
date: 2018-05-19 10:19:30
categories: Java
---

ThreadLocal 用来提供线程内部的局部变量的，并非是用于解决数据共享的，数据共享海兽需要使用关键字 synchronsize，上一篇笔记了解了 ThreadLocal 的基础使用，这篇笔记继续了解 ThreadLocal 的使用和实现原理。

# ThreadLocal 主要方法

ThreadLocal 对外提供的方法只有四个：

| 方法 | 用途 |
| --- | --- |
| public void set(T value) | 设置当前线程的线程局部变量的值 |
| public T get() | 返回当前线程所对应的线程局部变量 |
| public void remove() | 将当前线程局部变量的值删除，该方法是JDK 5.0新增的方法，可以加快内存回收的速度 (因为在线程结束的时候，对应该线程的局部变量是自动被垃圾机制自动回收的) |
| protected Object initialValue() | 返回该线程局部变量的初始值，是一个延迟调用方法，在线程第1次调用get()或set(Object)时才执行，并且仅执行1次 |

<!-- more  -->

# ThreadLocal 实现原理

首先我们要了解的是 ThreadLocal 里面的内部静态类 ThreadLocalMap，具体这个类里面怎么实现的这里不去深究，ThreadLocalMap 类里面相关的方法如下图所示：

![IMAGE](Java-多线程之ThreadLocal(二)/20180521133228.jpg)

从 ThreadLocalMap 构造方法和 set 方法可以看出 ThreadLocalMap 作为数据结构存值时是以 ThreadLocal 对象作为 key 的，：

```java
ThreadLocalMap(ThreadLocal<?> firstKey, Object firstValue) {
    // 具体实现
}

private void set(ThreadLocal<?> key, Object value) {
    // 具体实现
}
```