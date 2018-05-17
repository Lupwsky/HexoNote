---
title: Java-多线程之join
date: 2018-05-17 09:39:18
categories: Java
---

在一个线程生成并起动了另一个子线程，可以将第一个线程理解为主线程，另一个子线程理解为子线程，如果子线程里要进行大量的耗时的运算，主线程往往将于子线程之前结束，但是如果主线程需要用到子线程的处理结果，也就是主线程需要等待子线程执行完成之后再结束，这个时候就要用到 join 方法了，join 方法可以使主线程等待子线程终止后再继续执行后面调用 join 方法后面的代码。

# join 方法的使用

创建 JoinThreadB 线程类，代码如下：

```java
public class JoinThreadB extends Thread {
    @Override
    public void run() {
        super.run();
        System.out.println("线程 B 开始");
        try {
            System.out.println("线程 B 开始休眠 1 秒");
            sleep(1000);
            System.out.println("线程 B 开始休眠 1 秒");
            sleep(1000);
            System.out.println("线程 B 开始休眠 1 秒");
            sleep(1000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println("线程 B 结束");
    }
}
```

<!-- more -->

创建 JoinThreadA 线程类，在里面创建 JoinThreadB 的实例后并启动线程后，调用 JoinThreadB 的 join 方法，代码如下：

```java
public class JoinThreadA extends Thread {
    @Override
    public void run() {
        super.run();
        System.out.println("线程 A 开始");
        JoinThreadB threadB = new JoinThreadB();
        threadB.start();
        try {
            threadB.join(); // 不会再往下面执行，等待线程 B 执行完成或者出现异常倒是线程退出
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        System.out.println("线程 A 结束");
    }
}
```

创建 main 方法，创建 JoinThreadA 的实例并启动线程：

```java
public class Main {
    public static void main(String[] args) {
        JoinThreadA joinThreadA = new JoinThreadA();
        joinThreadA.start();
    }
}
```

运行的结果如下：

```text
线程 A 开始
线程 B 开始
线程 B 开始休眠 1 秒
线程 B 开始休眠 1 秒
线程 B 开始休眠 1 秒
线程 B 结束
线程 A 结束
```

# join 方法的源码

join 方法的实现是通过 等待/通知消息机制来实现的，看一看 join 方法的源码便知道了。

```java
public final synchronized void join(long millis) throws InterruptedException {
    long base = System.currentTimeMillis();
    long now = 0;

    if (millis < 0) {
        throw new IllegalArgumentException("timeout value is negative");
    }

    if (millis == 0) {
        while (isAlive()) {
            wait(0);
        }
    } else {
        while (isAlive()) {
            long delay = millis - now;
            if (delay <= 0) {
                break;
            }
            wait(delay);  // 多次调用 wait 方法，新设置的时间覆盖先前设置的时间
            now = System.currentTimeMillis() - base;
        }
    }
}
```

可以看到源码里面调用的是 wait 方法实现的，这里还要注意的是 wait 方法和 this.wait 方法的区别，wait 方法在这里是 CurrentThread.getThread().wait()，表示调用的是主线程的 wait 方法，让主线程处于等待状态。

# join 后是如何唤醒的

join 方法本身是使用了 synchronized 修饰符的，是加在方法上面的，意味着获取了当前线程对象的锁，然后继续发现里面的代码调用了 wait，意味着我们先锁，再释放，等待唤醒。查看官方文档，以线程对象作为锁的：After run() finishes, notify() is called by the Thread subsystem，当线程运行结束 (包括出现异常) 的时候，notify 是被线程的子系统自动调用的。
