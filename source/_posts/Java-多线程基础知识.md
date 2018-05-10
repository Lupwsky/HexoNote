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

1. 继承 Thread 类，Thread 是西安了 Runnable 接口，缺点是不能多继承
2. 实现 Runnable 接口

# 多次调用 start 方法

Thread 多次调用 start 方法，会出现 java.lang.IllegalThreadStateException 异常
![IMAGE](Java-多线程基础知识/20180510134836.png)