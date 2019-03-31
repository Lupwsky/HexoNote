---
title: Java-多线程之守护线程
date: 2019-03-02 17:08:31
categories: Java
---

# 守护线程

守护线程是一种特殊的线程, 当进程中不存在非守护线程了, 则守护线程自动销毁, 典型的线程就是垃圾回收线程, 当进行中所有的非守护线程不存在了 (程序就会结束), 垃圾回收线程也就没有必要存在了

# 非守护线程实例

创建一个线程, 示例代码如下:

```java
public class MThread extends Thread{
    @Override
    synchronized public void run() {
        super.run();
        try {
            for (int i = 0; i < 10000; i++) {
                sleep(1000);
                if (interrupted()) {
                    throw new InterruptedException();
                }
                System.out.println("由线程" + currentThread().getName() + "执行");
            }
        } catch (InterruptedException e) {
            System.out.println("线程" + currentThread().getName() + "中断了");
            return;
        }
        System.out.println("由线程" + currentThread().getName() + "执行，这个操作在循环体的外面");
    }
}
```

<!-- more -->

在 main 方法中启动这个线程, 示例代码如下:

```java
public static void main(String[] args) {
    Logger logger = LogManager.getLogger(Main.class.getName());
    MThread mThread = new MThread();
    Thread thread1 = new Thread(mThread, "A");
    thread1.start();
    try {
         Thread.sleep(5000);
    } catch (InterruptedException e) {
         e.printStackTrace();
    }
    logger.debug("main方法执行结束");
}
```

测试的输出结果如下:

```text
由线程A执行
由线程A执行
由线程A执行
由线程A执行
main方法执行结束
由线程A执行
由线程A执行
由线程A执行
由线程A执行
由线程A执行
```

可以看到在主线程执行完成后子线程并没有结束运行, 还在执行输出语句输出日志

# 守护线程实例

在 main 方法中执行这个线程, 启动线程前使用 Thread.setDaemon(true) 将线程设置成守护线程, 示例代码如下:

```java
public static void main(String[] args) {
    Logger logger = LogManager.getLogger(Main.class.getName());
    MThread mThread = new MThread();
    Thread thread1 = new Thread(mThread, "A");
    thread1.setDaemon(true);
    thread1.start();
    try {
        Thread.sleep(5000);
    } catch (InterruptedException e) {
        e.printStackTrace();
    }
    logger.debug("main方法执行结束");
}
```

测试的输出结果如下所示:

```text
由线程A执行
由线程A执行
由线程A执行
由线程A执行
main方法执行结束
```

可以看到在主线程结束后, 守护线程也结束运行了, 没有接着输出日志了