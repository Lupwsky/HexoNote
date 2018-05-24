---
title: Java-定时器Timer类和TimerTask类
date: 2018-05-23 19:03:25
categories: Java
---

Timer 类负责计划任务的功能，也即指定的时间开始执行某个任务。Timer 类的作用只是用于设置计划任务，对任务做排期，而任务的封装类则通过TimerTask完成。执行计划任务的代码要放入 TimerTask 的子类中，因为 TimerTask 为抽象类，具体功能均由子类处理，而 TimerTask 实现 Runnable 接口，Timer 中 TimerThread 类继承于 Thread 类。

# Timer 类

Timer 类对外的方法如下图所示：

<img src="Java-定时器Timer类和TimerTask类/20180523194306_435_360.jpg" width="400" height="350"/>

<!-- more -->

Timer 类本身没有继承于 Thread，在他的内部维护了两个对象的实例，TaskQueue 类的实例和 TimerThread 的实例，者两个类都是 Timer 的内部类，其中 TimerThread 继承于 Thread，Timer 类实现的底层实现还是使用线程来实现的，先了解 Timer 的构造方法和常用方法的使用。

## 构造方法

构造方法最多出现两个参数，参数 name 表示线程的名称，isDaemon 表示是否位守护线程， 各个构造方法的源码如下：

```java
/**
 * 当前线程名的编号，一个线程安全的 AtomicInteger 类
 */
private final static AtomicInteger nextSerialNumber = new AtomicInteger(0);
private static int serialNumber() {
    return nextSerialNumber.getAndIncrement();
}

// Timer(boolean isDaemon)
public Timer(boolean isDaemon) {
    this("Timer-" + serialNumber(), isDaemon);
}

// Timer(String name)
public Timer(String name) {
    thread.setName(name);
    thread.start();
}

// Timer(String name, boolean isDaemon)
public Timer(String name, boolean isDaemon) {
    thread.setName(name);
    thread.setDaemon(isDaemon);
    thread.start();
}
```

## schedule(TimerTask, long/Date) 方法

在指定的时间 (秒数) 或者指定的日期执行一次任务，如果是只当日期，如果当前日期已经超过指定的日期，就会立即执行。一个 Timer 可以多次调用 schedule 方法去执行不同的 TimerTask，执行 TimerTask 的是按顺序执行的。

```java
public class Main {
    public static void main(String[] args) {
        // 注意这里不能设置成守护线程，否则在主线程结束后，子线程也会退出导致计划任务不能再执行
        // Timer timer = new Timer(true);
        Timer timer = new Timer();
        timer.schedule(new TimerTask() {
            @Override
            public void run() {
                System.out.println("执行任务1");
            }
        }, 1000);

        timer.schedule(new TimerTask() {
            @Override
            public void run() {
                System.out.println("执行任务2");
            }
        }, 2000);
    }
}
```

上面的例子会在第一个任务执行完成后再过两秒执行第二个任务，而不是再第一秒执行第一个任务，第二秒执行第二个任务，因为执行 TimerTask 的是按顺序执行的。

## schedule(TimerTask, long/Date, long) 方法

再指定的日期或时间后按指定的周期时间无限循环执行某个任务。第二个参数为第一次执行任务的时间，第三个参数为循环时间隔的时间，如果当前时间超过指定的时间，那么第一任务会立即执行，之后按照指定的时间循环执行。用法和 schedule(TimerTask, long/Date) 方法类似，这里不再贴出具体使用的相关代码。

## schuduleAtFixedRate(TimerTask, long/Date, long) 方法

schuduleAtFixedRate() 方法和 schedule() 使用方法一样，但是不同的是 schedule() 执行 TimerTask 的时候是按顺序执行的，第二个和任务必须等待第一个任务执行完成后才会执行，而 schuduleAtFixedRate() 只要符合任务的执行时间，不管上一次任务是否执行完成都会执行。

## cancel() 方法

调用该方法后会清空所有的任务，并且调用了 cancle() 方法后，再调用 schedule() 方法则会出现异常 new IllegalStateException("Timer already cancelled.") 异常，如果在调用 cancel() 方法时最后一个任务正在等待执行，则会立即执行。

# Timer 类的实现原理

前面已经说过了 Timer 类本身没有继承于 Thread，在他的内部维护了两个对象的实例，TaskQueue 类的实例和 TimerThread 的实例，者两个类都是 Timer 的内部类，其中 TimerThread 继承于 Thread，Timer 类实现的底层实现还是使用线程来实现的。

```java
public class Timer {
    private final TaskQueue queue = new TaskQueue();
    private final TimerThread thread = new TimerThread(queue);
    // ...
}
```

查看源码，所有的 schedule() 方法最终调用的是 sched(TimerTask task, long time, long period) 方法，源码如下：

```java
public class Timer {
    // ...

    private void sched(TimerTask task, long time, long period) {
        if (time < 0) throw new IllegalArgumentException("Illegal execution time.");

        if (Math.abs(period) > (Long.MAX_VALUE >> 1))
            period >>= 1;

        synchronized(queue) {
            // newTasksMayBeScheduled 的值如果为 false 会抛出这个异常，调用 cancel() 方法后这个值会被设置为 fasle
            // 所以调用 cancel() 方法后再次调用 schedule() 的任何方法都会抛出这个异常
            if (!thread.newTasksMayBeScheduled) throw new IllegalStateException("Timer already cancelled.");

            synchronized(task.lock) {
                if (task.state != TimerTask.VIRGIN)
                    throw new IllegalStateException("Task already scheduled or cancelled");
                // 设置任务的执行时间和循环间隔时间
                task.nextExecutionTime = time;
                task.period = period;
                // 设置任务的状态为已经执行
                task.state = TimerTask.SCHEDULED;
            }

            // 将任务加入队列中
            queue.add(task);
            if (queue.getMin() == task)
                // 唤醒线程开始执行任务，由于在调用构造方法时就已经启动了线程执行了线程的 mainLoop() 方法，之前介绍 Timer 的构造方法贴出的源码可以看见有调用线程的 start() 方法，
                // 同时 newTasksMayBeScheduled 的值再创建线程的时候默认的初始值为 true，所以线程时处于等待状态的，具体的可以查看 TimerThread 里面 mainLoop() 方法。
                queue.notify();
        }
    }

    // cancel() 方法会将 newTasksMayBeScheduled 的值设置为 false，并清空队列，
    // 同时会唤醒线程，如果线程有调用 wait() 方法而处于等待状态，在调用 cancel() 方法后会立即执行，
    // 还有一个要注意的地方，如果某个任务正在执行，是不会受 cancel() 影响的，原因时 cancel() 方法和线程执行任务都需要 queue 锁，
    // 正在执行的任务已经获取到 queue 锁，导致 cancel() 方法在执行的时候需要等到 queue 锁被释放次啊能执行，所以 cancel() 方法对已经执行的任务没有任何影响。
    public void cancel() {
        synchronized(queue) {
            thread.newTasksMayBeScheduled = false;
            queue.clear();
            queue.notify();
        }
    }
    // ...
}
```

## TaskQueue

每一个 TimerTask 都会加入 TaskQueue 队列里面去维护，里面定义了将 TimerTask 加入对列，移除队列的各种维护的方法，TaskQueue 类源码比较简单，比较要注意的一点是，这个队列最多存放 128 个任务，如下所示：

```java
class TaskQueue {
    private TimerTask[] queue = new TimerTask[128];
    // ...
}
```

## TimerThread

TimerThread 继承于 Thread，最主要了解的是 mainLoop() 方法，这个方法轮询队列里面的任务来一个一个的执行这些任务，知道队列里面的任务执行完成或者再调用了 TimerTask 的 cancel() 方法，cancel() 方法会清楚 Timer 队列中的所有 TimerTask。

```java
class TimerThread extends Thread {
    boolean newTasksMayBeScheduled = true;
    private TaskQueue queue;

    TimerThread(TaskQueue queue) {
        this.queue = queue;
    }

    public void run() {
        try {
            mainLoop();
        } finally {
            // 线程结束清空队列，例如调用了 Timer 的 cancal() 方法，或者出现异常
            synchronized(queue) {
                newTasksMayBeScheduled = false;
                queue.clear();
            }
        }
    }

    private void mainLoop() {
        while (true) {
            try {
                TimerTask task;
                boolean taskFired;
                synchronized(queue) {
                    // 队列为空，但是有任务正在执行性，线程进入等待状态
                    while (queue.isEmpty() && newTasksMayBeScheduled) queue.wait();
                    // 队列为空，没有正在执行的任务，跳出循环，结束当前线程
                    if (queue.isEmpty()) break;

                    // 队列不为空，获取第一个需要执行的任务
                    long currentTime, executionTime;
                    task = queue.getMin();
                    synchronized(task.lock) {
                        // 如果这个任务调用了 TimerTask 的 cancel() 方法，从队列中移除这个任务并继续下一个循环
                        if (task.state == TimerTask.CANCELLED) {
                            queue.removeMin();
                            continue;
                        }
                        currentTime = System.currentTimeMillis();
                        // 获取任务下一次执行的时间
                        executionTime = task.nextExecutionTime;
                        // 如果当前时间超过指定执行任务的时间，并且这个只执行一次，从队列中先移除
                        if (taskFired = (executionTime <= currentTime)) {
                            // 通过设置的重复执行的任务的时间间隔是否为 0 判断这个任务是否是一个循环执行的任务
                            if (task.period == 0) {
                                queue.removeMin();
                                // 标记任务的状态为已执行
                                task.state = TimerTask.EXECUTED;
                            } else {
                                // 如果是玄幻执行的任务，重新设置这个任务下一次执行的时间
                                queue.rescheduleMin(task.period < 0 ? currentTime   - task.period : executionTime + task.period);
                            }
                        }
                    }
                    // 如果当前时间没有超过指定的时间，等待这个时间继续执行这个线程，注意这里使用的是 wait() 方法，
                    // 通常情况下就和之前所说的执行 TimerTask 的是按顺序执行的，更详细一点是按照 queue 队列里面的顺序执行的，
                    // 如果没有获取到 queue 锁，然后调用 notify() 方法，会一直再执行。
                    if (!taskFired) queue.wait(executionTime - currentTime);
                }
                // 执行任务
                if (taskFired) task.run();
            } catch(InterruptedException e) {
            }
        }
    }
}
```