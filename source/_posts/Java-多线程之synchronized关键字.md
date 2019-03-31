---
title: Java-多线程之synchronized关键字
date: 2019-03-02 17:17:30
categories: Java
---

# 非数据共享

线程都有自己的独立的成员变量或者这些需要处理的变量属于方法的私有变量, 就不会有数据共享的情况, 如下例子, 需要处理的数据是方法的私有变量, num为addNum的私有变量, 虽然线程使用的是同一个 HasSelfPrivateNum 实例对象, 但是不会有线程安全问题, 示例代码如下:

```java
public class HasSelfPrivateNum {
    void addNum(String username) {
        int num = username.equals("a") ? 100 : 200;
        System.out.println("num = " + num);
        try {
            Thread.sleep(2000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }
}
```

<!-- more -->

线程一示例代码:

```java
public class Thread1 extends Thread {
    private HasSelfPrivateNum hasSelfPrivateNum;
    public Thread1(HasSelfPrivateNum hasSelfPrivateNum) {
        this.hasSelfPrivateNum = hasSelfPrivateNum;
    }

    @Override
    public void run() {
        super.run();
        hasSelfPrivateNum.addNum("a");
    }
}
```

线程二示例代码:

```java
public class Thread2 extends Thread {
    private HasSelfPrivateNum hasSelfPrivateNum;
    public Thread2(HasSelfPrivateNum hasSelfPrivateNum) {
        this.hasSelfPrivateNum = hasSelfPrivateNum;
    }

    @Override
    public void run() {
        super.run();
        hasSelfPrivateNum.addNum("b");
    }
}
```

在 main 方法中允许, 两个线程里面使用的的是私有数据, 在并发情况下不会相互影响, 这些数据都是非共享数据:

```java
public static void main(String[] args) {
    Logger logger = LogManager.getLogger(Main.class.getName());
    HasSelfPrivateNum hasSelfPrivateNum = new HasSelfPrivateNum();
    Thread thread1 = new Thread1(hasSelfPrivateNum);
    Thread thread2 = new Thread2(hasSelfPrivateNum);
    thread1.start();
    thread2.start();
    logger.debug("main方法结束");
}
```

# 共享数据造成的线程安全问题和脏读问题

上面的例子中, 两个线程访问的是同一个对象, 但是num是一个非共享数据, 所以不会有线程安全问题, 如果将 num 设置为类的成员变量, 两个线程同时访问这个变量就会出现线程安全问题, 实例如下：

```java
public class HasSelfPrivateNum {
    private int num;
    void addNum(String username) {
        if (username.equals("a")) {
            num = 100;
            try {
                Thread.sleep(2000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            System.out.println("num = " + num);
        } else {
            num = 200;
            System.out.println("num = " + num);
        }
    }
}
```

输出结果如下:

```text
num = 200
main方法结束
num = 200
```

可以看到, 理论上我想线程一执行后后输出的 num = 100, 线程二执行后输出的 num = 200, 但是现在数据出现的了脏读, 两次 num 的值均为 200, 多个线程因为执行同一段代造成了数据不安全, 会导致程序出现逻辑上的问题

# synchronized 关键字解决线程安全问题

synchronized 是一种同步锁, 经过 synchronized 修饰的代码在一个时间内只能有一个线程得到执行, 可以用来解决线程安全的问题, 它修饰的对象有以下几种

* 修饰一个代码块, 被修饰的代码块称为同步语句块, 其作用的范围是大括号 {} 括起来的代码, 作用的对象是调用这个代码块的对象
* 修饰一个方法, 被修饰的方法称为同步方法, 其作用的范围是整个方法, 作用的对象是调用这个方法的对象
* 修饰一个静态的方法, 其作用的范围是整个静态方法, 作用的对象是这个类的所有对象
* 修饰一个类, 其作用的范围是 synchronized 后面括号括起来的部分, 作用主的对象是这个类的所有对象

上面的例子使用 synchronized 关键字修饰后的代码如下, 更改后再次执行, 实例代码如下:

```java
public class HasSelfPrivateNum {
    private int num;
    synchronized void addNum(String username) {
        if (username.equals("a")) {
            num = 100;
            try {
                Thread.sleep(2000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            System.out.println("num = " + num);
        } else {
            num = 200;
            System.out.println("num = " + num);
        }
    }
}
```

测试的结果输出如下:

```text
num = 100
main方法结束
num = 200
```

可以看到, 输出的结果使我们期望的结果

# 多个对象多个锁

在上一个的例子中, 使用了 synchronized 关键字解决同步的问题, 可以发现这个例子中, 线程的调用时顺序执行的 (虽然 thread1 比 thread2 先执行, 但是thread1不一定就比 thread2 先执行), thread2 在 thread1 休眠了两秒, 等待 thread1 打印语句后才指定打印语句的, 是因为 synchronized 取得的是一个对象锁 (都是针对一个相同的实例), 而不是取得某个方法或者代码的锁, 那个线程先执行带有 synchronized 的方法, 就会先取得锁, 那么其他线程只能是等待状态, 如果有多个对象就会有多个锁, 实例如下(使用多个对象在上面的例子都不会有线程安全问题, 这里只是举例用):

```java
public static void main(String[] args) {
    Logger logger = LogManager.getLogger(Main.class.getName());
    Thread thread1 = new Thread1(new HasSelfPrivateNum());
    Thread thread2 = new Thread2(new HasSelfPrivateNum());
    thread1.start();
    thread2.start();
    logger.debug("main方法结束");
}
```

输出的结果如下, thread2 线程不需要等待 thread1 执行完打印语句才会执行, 两个线程取得的对象锁是不同的:

```text
num = 200
main方法结束
num = 100
```

知道了 synchronized 获取的是对象锁, 就可以知道只要是有线程获取了锁, 其他的线程在调用其他加了关键字 synchronized 的关键字的方法就需要等待, 但是没有使用关键字的 synchronized 的方法则可以不需要等待就可以执行

# synchronized 和锁重入功能

* synchronized 锁重入功能, 当一个线程获取到锁后, 这个线程再次获取锁 (这个锁还没有被释放) 任然可以获取到, 即在调用了 synchronized 方法的里面在次调用有 synchronized 关键的方法也可以获取到锁, 如果不能重入锁, 在 synchronized 的方法里面调用一个 synchronized 方法, 就会造成死锁问题
* 如果线程出现了异常, 其持有的锁都会被释放
* 同步不具有继承性

# synchronized 代码块

synchronized 声明的方法也是有弊端的, 如果A线程调用一个同步方法执行一个长时间任务, 那么B线程就必须等待较长时间, 这个时候就可以使用 synchronized 同步语块来解决这类问题, synchronized 代码块的使用方法不是在方法前添加关键字, 而是在方法里面添加代码块中使用如下格式代码:

```java
private void xxxxx() {
    // TODO 逻辑代码 1
    synchronized(this) {
        // TODO 逻辑代码 2
    }
    // TODO 逻辑代码 3
}
```

其中逻辑代码 1 和逻辑代码3会异步执行, 逻辑代码 2 则会同步执行, 如果逻辑代码 1 和逻辑代码 3 是耗时操作, 那么使用代码 synchronized 块的方法比使用 synchronized 方法的效率会高很多