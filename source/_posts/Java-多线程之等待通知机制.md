---
title: Java-多线程之等待通知机制
date: 2018-05-15 22:13:45
categories: Java
---

线程和线程之间通过同步来实现数据的共享，线程和线程之间也可以相互通信和协作，在 Java 的线程中使用 wait/notify 的方式实现线程之间的通信。

# 等待/通知机制

在 Java 中，使用 Object 中的 wait 和 notify 方法来实现等待/通知 (要注意对象 A 调用的 wait 方法只能被对象 A 的 notify 方法唤醒, 不能被其他的对象的 notify 方法唤醒)

## wait 方法

* wait 方法是 Object 类的方法，使当前执行代码的线程进入等待状态，方法在调用的地方就会停止代码的执行，直到收到通知或者被中断；
* wait 方法必须在同步方法或者同步代码块中调用，否则出现 IllegalMotitorStateException 异常；
* wait 方法执行后，线程会立即释放锁，线程将再次和其他线程竞争获取锁。

<!-- more -->

## notify 方法

* notify 方法是 Object 类的方法，用于唤醒处于等待状态的线程，如果有多个线程处于等待状态，随机唤醒一个线程执行；
* notify 方法必须在同步方法或者同步代码块中调用，否则出现 IllegalMotitorStateException 异常；
* notify 方法执行后，线程会不会立即释放锁，线程在同步方法或同步代码块执行完成后才会释放锁，线程将再次和其他线程竞争获取锁。

## notifyAll 方法

notifyAll 方法可以使使用同一个对象调用 wait 方法的线程全部唤醒，进入可运行状态，优先级高的线程有大概率会先执行。

## 线程在等待状态下调用 interrupt 方法

当线程处于 wait 状态时，调用线程的 interrupt 方法出现 InterruptException 异常。

## 线程释放锁的情况

* 执行完同步代码就会立即释放锁；
* 执行同步代码出现异常会立即释放锁；
* 在同步代码中调用 wait 方法会立即释放锁。

## 通知过早

在线程的 wait 方法调用之前就调用了 notify 方法，这种情况称之为通知过早，通知过早，会导致 wait 的线程永远无法被唤醒，使程序出现错误。

# wait 和 sleep 的区别

sleep 方法需要指定等待的时间，它可以让当前正在执行的线程在指定的时间内暂停执行，进入阻塞状态，该方法既可以让其他同优先级或者高优先级的线程得到执行的机会，也可以让低优先级的线程得到执行机会。但是 sleep 方法不会释放锁标志，也就是说如果有 synchronized 同步块，其他线程仍然不能访问共享数据。

wait 方法会释放对象的锁，当调用某一对象的 wait 方法后，会使当前线程暂停执行，并将当前线程放入对象等待池中，直到调用了相同对象的 notify 方法后才会被唤醒。

# 使用通知/等待机制实现生产者/消费者模式

生产者/消费者模式是线程等待/通知机制的一种经典应用, 测试用例如下, 创建一个消费者线程, 一个生产者线程, 内容如下所示

```java
// 生产者线程
@Slf4j
public class ProviderThread extends Thread {
    private final String lock;

    public ProviderThread(String lock) {
        this.lock = lock;
    }

    @Override
    public void run() {
        boolean stop = false;
        while (!stop) {
            try {
                synchronized (lock) {
                    // notify 不是一调用就释放锁, 而是等待同步代码块执行完后才释放
                    // 因此 sleep 方法在这个例子里面不能在同步代码块中调用
                    // 否则容易出现消费者线程不会获取到锁而这个生产者线程一直获取到锁的情况
                    // 结果导致消费者不能获取到锁, 也就不能消费消息了
                    // sleep(5000);
                    lock.notify();
                    log.info("发布了一条消息...");
                }
                sleep(5000);
            } catch (InterruptedException e) {
                e.printStackTrace();
                stop = true;
            }
        }
    }
}

// 消费者线程
@Slf4j
public class ComsumerThread extends Thread {
    private final String lock;

    public ComsumerThread(String lock) {
        this.lock = lock;
    }

    @Override
    public void run() {
        boolean stop = false;
        while (!stop) {
            try {
                synchronized (lock) {
                    log.info("消费者 {} 正在等待消息", this.getName());
                    lock.wait();
                    log.info("消费者 {} 消费了一条消息", this.getName());
                }
            } catch (InterruptedException e) {
                e.printStackTrace();
                stop = true;
            }
        }
    }
}
```

在 main 方法中启动消费者和生产者, 注意生产者的 wait 方法和消费者的 notify 方法都是使用的同一个 lock 对象调用的, 示例如下

```java
public class SubscriptMain {
    public static void main(String[] args) {
        String lock = "LOCK";

        ComsumerThread comsumerThreadA = new ComsumerThread(lock);
        comsumerThreadA.setName("A");
        comsumerThreadA.start();

        ComsumerThread comsumerThreadB = new ComsumerThread(lock);
        comsumerThreadB.setName("B");
        comsumerThreadB.start();

        ProviderThread providerThread = new ProviderThread(lock);
        providerThread.start();
    }
}
```

控制台输出的部分日志如下所示

```text
[2019-05-30 14:23:54][INFO ][com.lupw.guava.thread.ComsumerThread] - 消费者 A 正在等待消息
[2019-05-30 14:23:55][INFO ][com.lupw.guava.thread.ComsumerThread] - 消费者 B 正在等待消息
[2019-05-30 14:23:55][INFO ][com.lupw.guava.thread.ProviderThread] - 发布了一条消息...
[2019-05-30 14:23:55][INFO ][com.lupw.guava.thread.ComsumerThread] - 消费者 A 消费了一条消息
[2019-05-30 14:23:55][INFO ][com.lupw.guava.thread.ComsumerThread] - 消费者 A 正在等待消息
[2019-05-30 14:23:57][INFO ][com.lupw.guava.thread.ProviderThread] - 发布了一条消息...
[2019-05-30 14:23:57][INFO ][com.lupw.guava.thread.ComsumerThread] - 消费者 B 消费了一条消息
[2019-05-30 14:23:57][INFO ][com.lupw.guava.thread.ComsumerThread] - 消费者 B 正在等待消息
```