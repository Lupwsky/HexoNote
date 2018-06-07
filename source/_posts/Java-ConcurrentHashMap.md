---
title: Java-ConcurrentHahsMap实现原理（JDK 1.8）- 未完成
date: 2018-05-31 17:51:32
categories: Java
---

ConcurrentHashMap 是一种线程安全且高效的 HashMap，HashMap 是非线程安全的，在多线程情况下使用 HashMap 的 put 操作 (发生在扩容的时) 可能会引起死循环。HashTable 虽然是线程安全的，但是其效率低下。ConcurrentHashMap 就可以解决上面出现的问题。ConcurrentHashMap 在 JDK 1.8 中相较之前的版本改动较大，使用了全新的 CAS 算法实现，本文我主要看的还是 JDK 1.8 中的源码，和之前版本的区别就简单的了解一下。

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

# ConcurrentHashMap 1.7

ConcurrentHashMap 在 JDK 1.8 中更改很大，摒弃了 Segment (段) 的概念，而是启用了一种全新的方式实现，利用 CAS 算法，底层任然使用数组结构加链表的方式实现 (在 JDK1.8 红黑树)。在 JDK 1.8 中为了做到并发，又增加了 TreeBin，Traverser 等对象内部辅助类。

## 锁分段技术

鉴于 HashTable 使用只是用一把锁来实现同步，在存数据的时候，获取数据，移除数据都会阻塞直到锁释放才能进行，导致在高并发环境下读写数据效率低下。ConcurrentHashMap 则使用了锁分段技术，在 ConcurrentHashMap 里面存在多把锁，每一把锁都作用于其中的一段数据，当在多线程的环境下时，多个线程访问不同的数据段就不不会出现像 HashTable 那样去竞争同一把锁，从而提高并发访问效率。ConcurrentHashMap 在存储数据的时候就将数据分段存储，然后给每一段数据都会添加一把锁。

## 结构图

首先看一看 JDK 1.7 ConcurrentHashMap 结构图，如下：

<img src="Java-ConcurrentHahsMap/20180601101044.jpg" width="950" height="450"/>

一个 ConcurrentHashMap 由 Segment 数组和结构和 Map.Entry (Node 类继承实现 Map.Entry 类) 数组组成。Senment (段) 类是一种可重入锁 (ReentrantLock)，通过使用 Segment 将 ConcurrentHashMap 划分为不同的部分，ConcurrentHashMap 就可以使用不同的锁来控制对哈希表的不同部分的修改，从而允许多个修改操作并发进行，由于在 JDK1.8 中已经不在使用锁分段技术，JDK 1.7 的源码这里就不贴出， Segment 类在 1.8 中定义如下，仅仅是为了兼容序列化而声明：

```java
/**
 * 这是 JDK 1.8 中 Segment 的源码，相比之前的版本这是一个精简版，
 * 仅仅为了序列化兼容性而声明
 */
static class Segment<K,V> extends ReentrantLock implements Serializable {
    private static final long serialVersionUID = 2249069246763182397L;
    final float loadFactor;
    Segment(float lf) { this.loadFactor = lf; }
}
```

# ConcurrentHashMap 1.8

## CAS 算法

CAS (比较与交换，Compare and swap)，CAS 是一种无锁算法，CAS 有 3 个操作数，内存值 V，旧的预期值 A，要修改的新值 B，当且仅当预期值 A 和内存值 V 相同时，将内存值 V 修改为 B，否则什么都不做。详细可以参考以下资料：

* 非阻塞同步算法与 CAS(Compare and Swap) 无锁算法：[http://www.cnblogs.com/Mainz/p/3546347.html](http://www.cnblogs.com/Mainz/p/3546347.html)
* 深入理解 CAS 算法原理：[https://cloud.tencent.com/developer/article/1078710](https://cloud.tencent.com/developer/article/1078710)
* 乐观锁的一种实现方式—CAS：[http://www.importnew.com/20472.html](http://www.importnew.com/20472.html)* 
* Java CAS 理解：[https://mritd.me/2017/02/06/java-cas/](https://mritd.me/2017/02/06/java-cas/)

## 结构图

ConcurrentHashMap 在 JDK 1.8 中使用了 CAS 算法实现，进一步的提高并发性，摒弃了锁分段技术，采用的是直接使用一个数组 + 链表 + 红黑树的结构，和 HashMap 的结构图是一样的，其结构图如下：

<img src="Java-ConcurrentHahsMap/20180531121839.jpg" width="700" height="450"/>

## 哈希桶数组元素 Node 类

ConcurrentHashMap 的结构和 HashMap 是类似的，其哈希桶数组也是 Node 类型的数组，但是和 HashMap 的数组有一些区别：

第一点是 val 和 next 字段都加上了 volatile 关键字，使得这两个字段对其他线程是可见的。

```java
volatile V val;
volatile Node<K,V> next;
```

第二点是 val 值不能被设置。

```java
public final V setValue(V value) {
    throw new UnsupportedOperationException();
}
```

第三点是新增了 find() 方法用于支持 map.get() 获取数据。

```java
// h：键在散列后的值，k：键
Node<K,V> find(int h, Object k) {
    Node<K,V> e = this;
    if (k != null) {
        do {
            K ek;
            if (e.hash == h && ((ek = e.key) == k || (ek != null && k.equals(ek))))
                return e;
        } while ((e = e.next) != null);
    }
    return null;
}
```

## 哈希方法

ConcurrentHashMap 的哈希方法和 HashMap 的方法是一样的。

```java
// h 是 key 的 hashCode
static final int HASH_BITS = 0x7fffffff; // usable bits of normal node hash
static final int spread(int h) {
    return (h ^ (h >>> 16)) & HASH_BITS;
}
```

## put 操作

ConcurrentHashMap 保存数据主要调用的是 putVal() 方法，主要实现如下：

```java
// hash表初始化或扩容时的一个控制位标识量。
// 负数代表正在进行初始化或扩容操作
// -1代表正在初始化
// -N 表示有N-1个线程正在进行扩容操作
// 正数或0代表hash表还没有被初始化，这个数值表示初始化或下一次进行扩容的大小
private transient volatile int sizeCtl;

final V putVal(K key, V value, boolean onlyIfAbsent) {
    if (key == null || value == null) throw new NullPointerException();
    int hash = spread(key.hashCode());
    int binCount = 0;
    for (Node<K,V>[] tab = table;;) {
        Node<K,V> f;
        // n：桶数组当前长度，i：这个 key 散列取模后的数组索引值，fh：key 的哈希值
        int n, i, fh;
        // table 还没有初始化，先初始化通数组
        if (tab == null || (n = tab.length) == 0) {
            tab = initTable();
        } else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
            // 如果哈希桶数组中没有保存这个 key，CAS 算法添加这个 key
            if (casTabAt(tab, i, null, new Node<K,V>(hash, key, value, null))) {
                break;
            }
        } else if ((fh = f.hash) == MOVED) {
            tab = helpTransfer(tab, f);
        } else {
            V oldVal = null;
            synchronized (f) {
                if (tabAt(tab, i) == f) {
                    if (fh >= 0) {
                        binCount = 1;
                        // 循环遍历将 key 添加到链表
                        for (Node<K,V> e = f;; ++binCount) {
                            K ek;
                            // 找到相同的 key，替换掉旧的值
                            if (e.hash == hash && ((ek = e.key) == key || (ek != null && key.equals(ek)))) {
                                oldVal = e.val;
                                // 根据 onlyIfAbsent 的值设置时候替换已经存在的 key 对应的值
                                if (!onlyIfAbsent) {
                                    e.val = value;
                                }
                                break;
                            }
                            // 保存新的 Node 的引用到上一个 Node 的 next 里面
                            Node<K,V> pred = e;
                            if ((e = e.next) == null) {
                                pred.next = new Node<K,V>(hash, key, value, null);
                                break;
                            }
                        }
                    } else if (f instanceof TreeBin) {
                        Node<K,V> p;
                        binCount = 2;
                        // 将新的 Node 插入到红黑树里面
                        if ((p = ((TreeBin<K,V>)f).putTreeVal(hash, key, value)) != null) {
                            oldVal = p.val;
                            if (!onlyIfAbsent) {
                                p.val = value;
                            }
                        }
                    }
                }
            }
            if (binCount != 0) {
                // 链表数量大于 8，转换成红黑树
                if (binCount >= TREEIFY_THRESHOLD) {
                    treeifyBin(tab, i);
                }
                if (oldVal != null) {
                    return oldVal;
                }
                break;
            }
        }
    }
    addCount(1L, binCount);
    return null;
}
```

## size 操作

JDK 1.7 中获取 ConcurrentHashMap 当前元素的大小需要将 Segment 类里面的 count (volatile 修饰的变量) 累加，在计算总元素大小的时候需要 put、remove 和 clean 操作全部锁住才能得到正确的值，但是这种效率是非常低下的，因此 JDK 1.7 中的做法是先采取两次不锁住 Segment 的方式计算 size 的大小，因为之前累加过的 count 变化的几率较小，如果连续两次都计算 size 时都有 count 发生了变化，采用锁定 Segment 的方法来计算 size。ConcurrentHashMap 使用在 size 前后比较 modCount 的值是否相等来判定是否有 count 发生了变化，应为所有的put、remove 和 clena 操作都会改动 modCount 的值。

JDK 1.8 中新增了一个mappingCount() 方法，两个方法都同时调用了，sumCount() 方法，sumCount() 只是返回当前元素的值，在多线程环境下这个统计值只是一个估计的值 ，所以 size() 和 mappingCount() 返回的也都是一个估计值。JDK 1.8 牺牲了获取元素数量的精度，来换取更高的并发效率。

```java
public int size() {
    long n = sumCount();
    return ((n < 0L) ? 0 : (n > (long) Integer.MAX_VALUE) ? Integer.MAX_VALUE :(int) n);
}

// 1.8 加入 
public long mappingCount() {
    long n = sumCount();
    return (n < 0L) ? 0L : n;
}

final long sumCount() {
    CounterCell[] as = counterCells; CounterCell a;
    long sum = baseCount;
    if (as != null) {
        for (int i = 0; i < as.length; ++i) {
            if ((a = as[i]) != null)
                sum += a.value;
        }
    }
    return sum;
}
```

# 参考资料

* JDK 1.7 HashTable 类和 ConcurrentHashMap 类的源码
* JDK 1.8 HashTable 类和 ConcurrentHashMap 类的源码
* 书籍-Java 并发编程的艺术
* 从ConcurrentHashMap的演进看Java多线程核心技术：[http://www.jasongj.com/java/concurrenthashmap/](http://www.jasongj.com/java/concurrenthashmap/)