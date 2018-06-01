---
title: Java-ConcurrentHahsMap
date: 2018-05-31 17:51:32
categories: Java
---

ConcurrentHashMap 是一种线程安全且高效的 HashMap，HashMap 是非线程安全的，在多线程情况下使用 HashMap 的 put 操作 (发生在扩容的时) 可能会引起死循环。HashTable 虽然是线程安全的，但是其效率低下。ConcurrentHashMap 就可以解决上面出现的问题。

# HashTable 效率低原因

我们先可以看看 HashTable 的源码中对 Map 进行操作的几个方法的定义，如下：

```java
// put 方法
public synchronized V put(K key, V value) {
    // ...
}

// remove 方法
public synchronized V remove(Object key) {
    // ...
}

// putAll 方法
public synchronized void putAll(Map<? extends K, ? extends V> t) {
    // ...
}
```

<!-- more -->

可以了解到奥到所有和 HashTable 操作有关的都是在方法前面添加 synchronized 关键来实现同步的，在方法前添加 synchronized 获取的时线程锁，所有同步的方法都要获取到这把锁才能对 Map 进行操作，在多线程环境下，其他的线程也要竞争这把锁，而且在调用了同步的 put() 方法时，其他的操作如获取数据也要等锁释放才能进行获取数据的操作。

# ConcurrentHashMap 的锁分段技术

鉴于 HashTable 使用只是用一把锁来实现同步，在存数据的时候，获取数据，移除数据都会阻塞直到锁释放才能进行，导致在高并发环境下读写数据效率低下。ConcurrentHashMap 则使用了锁分段技术，在 ConcurrentHashMap 里面存在多把锁，每一把锁都作用于其中的一段数据，当在多线程的环境下时，多个线程访问不同的数据段就不不会出现像 HashTable 那样去竞争同一把锁，从而提高并发访问效率。ConcurrentHashMap 在存储数据的时候就将数据分段存储，然后给每一段数据都会添加一把锁。

# ConcurrentHashMap 的实现原理

# 参考资料

* JDK 1.8 HashTable 类和 ConcurrentHashMap 的源码
* 书籍-Java 并发编程的艺术