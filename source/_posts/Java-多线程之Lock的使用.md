---
title: Java-多线程之Lock的使用
date: 2018-05-21 14:20:49
categories: Java
---

synchronized 可以解决多线程资源同步的问题，但是 synchronized 的性能是比较低效的，例如在一个线程获取到锁后，如果这线程阻塞了 (如IO操作)，由于使用 synchronized 获取到锁并发生阻塞的线程是不能够响应中断，所以其他线程就会一直等待下去，为此就需要有一种机制能够解决这种线程同步中无限等待的问题，使用 Lock 就可以解决类似的问题。

注：这里锁的性能差是相对于 Lock 的比较，但是在 Java 1.6 后，synchronize进行了很多优化，有适应自旋，锁消除，锁粗化，轻量级锁，偏向锁等等，导致 synchronize 的性能并不比 Lock 差。

# Lock 和 synchronized 的区别

1. synchronized 是 Java 语言内置的关键字，Lock 是一个类，通过这个类可以实现同步访问，一个是 JVM 层面的，一个是 JDK 层面的;
2. synchronized 可以加在方法上，也可以加在代码块中，Lock 只能在在代码块中;
3. synchronized 不需要手动释放锁，会自动释放对锁的占用，而 Lock 则必须要手动释放锁，如果没有主动释放锁，会出现死锁现象。

<!-- more -->

# Lock 类
