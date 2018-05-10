---
title: Java-多线程基础知识
date: 2018-05-10 13:30:28
categories: Java
---

# 什么是进程、线程

进程：操作系统上一次程序的运行，例如 Windows 上的 QQ.exe 程序
线程：在进程中独立运行的子任务，例如 QQ 程序里面运行的文件下载，音频等子任务，这些任务大多是在后台异步运行

# 多线程的优点

使用多线程可以在同一时间内运行更多的任务，最大限度的利用 CPU 资源处理更多的任务。

# Java 实现多线程

- 继承 Thread 类，Thread 是西安了 Runnable 接口，缺点是不能多继承
- 实现 Runnable 接口

<!-- more -->

# 多次调用 start 方法

Thread 多次调用 start 方法，会出现 java.lang.IllegalThreadStateException 异常，查看和调试 start 方法的源码如下图所示

![IMAGE](Java-多线程基础知识/20180510134836.png)

调试下源代码，Thread 类里面有一个 threadStatus 变量，当调用 start 时会首先判断这个值是否等于 0，如果等于 0，说明这个线程还未启动。在源码中可以看到，如果一个线程 start，调用的是原生的方法 start0，调用这个方法后会赋予 threadStatus 一个不为 0 的值，因此再次调用 start 方法时就会出现 IllegalThreadStateException 异常了。