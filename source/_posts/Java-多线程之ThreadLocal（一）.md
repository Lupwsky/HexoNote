---
title: Java-多线程之ThreadLocal(一)
date: 2018-05-17 21:42:42
categories: Java
---

ThreadLocal 提供了线程本地的实例，它与普通变量的区别在于，每个使用该变量的线程都会初始化一个完全独立的实例副本。ThreadLocal 变量通常被 private static 修饰。当一个线程结束时，它所使用的所有 ThreadLocal 相对的实例副本都可被回收。ThreadLocal 类为每个使用该变量的线程提供独立的变量副本，在本线程内部，它相当于一个"全局变量"，可以保证本线程任何时间操纵的都是同一个对象。因为每个 Thread 内有自己的实例副本，且该副本只能由当前 Thread 使用，因此使用 ThreadLocal 的线程并不存在共享变量的问题，它不是用来解决线程同步的问题的。

# 非线程安全的例子

先看一个非线程安全的例子，这个线程 A 和线程 B 共享同一个数据，但是由于没有同步，导致逻辑混乱。

LocalParam 类：

```java
public class LocalParam {
    private int num;

    public LocalParam() {
    }

    public int getNum() {
        return num;
    }

    public void setNum(int num) {
        this.num = num;
    }
}
```

<!-- more -->

ThreadA 类：

```java
public class ThreadA extends Thread {
    private LocalParam localParam;

    public ThreadA(LocalParam localParam) {
        this.localParam = localParam;
    }

    @Override
    public void run() {
        super.run();
        for (int i = 0; i < 10; i++) {
            localParam.setNum(localParam.getNum() + 1);
            System.out.println("ThreadA num = " + localParam.getNum());
        }
    }
}
```

ThreadB 类：

```java
public class ThreadB extends Thread {
    private LocalParam localParam;

    public ThreadB(LocalParam localParam) {
        this.localParam = localParam;
    }

    @Override
    public void run() {
        super.run();
        for (int i = 0; i < 10; i++) {
            localParam.setNum(localParam.getNum() + 1);
            System.out.println("ThreadB num = " + localParam.getNum());
        }
    }
}
```

Main 类：

```java
public class Main {
    private static LocalParam localParam = new LocalParam();
    public static void main(String[] args) {
        ThreadA threadA = new ThreadA(localParam);
        ThreadB threadB = new ThreadB(localParam);
        threadA.start();
        threadB.start();
    }
}
```

输出结果如下：

```text
Connected to the target VM, address: '127.0.0.1:52604', transport: 'socket'
Disconnected from the target VM, address: '127.0.0.1:52604', transport: 'socket'
ThreadA num = 2
ThreadB num = 2
ThreadA num = 3
ThreadB num = 4
ThreadA num = 5
ThreadA num = 7
ThreadA num = 8
ThreadA num = 9
ThreadA num = 10
ThreadA num = 11
ThreadB num = 6
ThreadA num = 12
ThreadB num = 13
ThreadA num = 14
ThreadB num = 15
ThreadB num = 16
ThreadB num = 17
ThreadB num = 18
ThreadB num = 19
ThreadB num = 20
```

本来期望的是输出 num 值按照 1 到 20 输出，但是结果输出都是乱的，这是由于非线程安全造成的结果，如果希望按照所想的额输出结果，就需要使用关键字 synchronized 来进行同步。

# 线程安全的例子

将上面的例子更改成线程安全的例子，为了简单，这里直接在 ThreadA 类和 ThreadB 的 run 方法前添加 synchronized 关键字，其他的代码不变，修改后如下。

ThreadA 类修改后的代码：

```java
public class ThreadA extends Thread {
    private LocalParam localParam;

    public ThreadA(LocalParam localParam) {
        this.localParam = localParam;
    }

    @Override
    public synchronized void run() {
        super.run();
        for (int i = 0; i < 10; i++) {
            localParam.setNum(localParam.getNum() + 1);
            System.out.println("ThreadA num = " + localParam.getNum());
        }
    }
}
```

ThreadB 类修改后的代码：

```java
public class ThreadB extends Thread {
    private LocalParam localParam;

    public ThreadB(LocalParam localParam) {
        this.localParam = localParam;
    }

    @Override
    public synchronized void run() {
        super.run();
        for (int i = 0; i < 10; i++) {
            localParam.setNum(localParam.getNum() + 1);
            System.out.println("ThreadB num = " + localParam.getNum());
        }
    }
}
```

运行结果如下：

```text
Connected to the target VM, address: '127.0.0.1:53617', transport: 'socket'
Disconnected from the target VM, address: '127.0.0.1:53617', transport: 'socket'
ThreadB num = 1
ThreadA num = 2
ThreadB num = 3
ThreadA num = 4
ThreadB num = 5
ThreadA num = 6
ThreadB num = 7
ThreadA num = 8
ThreadB num = 9
ThreadA num = 10
ThreadB num = 11
ThreadA num = 12
ThreadB num = 13
ThreadA num = 14
ThreadB num = 15
ThreadA num = 16
ThreadB num = 17
ThreadA num = 18
ThreadB num = 19
ThreadA num = 20
```

修改后 num 的值就是按照预想的结果输出了。

## 两个线程交替输出的问题

但是这里有一个问题，按照代码的逻辑，首先是线程 A 或者线程 B 中的一个线程获取锁之后，然后循环 10 次输出，在 run 方法完成后，然后另外一个线程获取锁，再循环 10 次进行输出，结果应该是下面类似，先由其中一个线程输出 1 到 10，再由另外一个线程输出 11 到 20。

```text
Connected to the target VM, address: '127.0.0.1:53617', transport: 'socket'
Disconnected from the target VM, address: '127.0.0.1:53617', transport: 'socket'
ThreadA num = 1
ThreadA num = 2
ThreadA num = 3
ThreadA num = 4
ThreadA num = 5
ThreadA num = 6
ThreadA num = 7
ThreadA num = 8
ThreadA num = 9
ThreadA num = 10
ThreadB num = 11
ThreadB num = 12
ThreadB num = 13
ThreadB num = 14
ThreadB num = 15
ThreadB num = 16
ThreadB num = 17
ThreadB num = 18
ThreadB num = 19
ThreadB num = 20
```

但是上面的例子确是交替输出的，为什么呢？输出结果如下：

```text
Connected to the target VM, address: '127.0.0.1:53617', transport: 'socket'
Disconnected from the target VM, address: '127.0.0.1:53617', transport: 'socket'
ThreadB num = 1
ThreadA num = 2
ThreadB num = 3
ThreadA num = 4
ThreadB num = 5
ThreadA num = 6
ThreadB num = 7
ThreadA num = 8
ThreadB num = 9
ThreadA num = 10
ThreadB num = 11
ThreadA num = 12
ThreadB num = 13
ThreadA num = 14
ThreadB num = 15
ThreadA num = 16
ThreadB num = 17
ThreadA num = 18
ThreadB num = 19
ThreadA num = 20
```

## 共享数据是局部变量同步无效的问题

如果将 Mian 类中的 lcoalParam 定义成局部的变量即使同步输出也是错误的，这又是为什么？

Main 方法中的 localParam 定义成局部变量：

```java
public class Main {
    public static void main(String[] args) {
        LocalParam localParam = new LocalParam();
        ThreadA threadA = new ThreadA(localParam);
        ThreadB threadB = new ThreadB(localParam);
        threadA.start();
        threadB.start();
    }
}
```

Main 方法中的 localParam 定义成局部变量，虽然同步处理，但是输出还是错误的：

```text
Connected to the target VM, address: '127.0.0.1:53815', transport: 'socket'
Disconnected from the target VM, address: '127.0.0.1:53815', transport: 'socket'
ThreadB num = 1
ThreadB num = 2
ThreadB num = 3
ThreadB num = 4
ThreadB num = 5
ThreadB num = 6
ThreadA num = 1
ThreadA num = 8
ThreadA num = 9
ThreadA num = 10
ThreadB num = 7
ThreadA num = 11
ThreadA num = 13
ThreadA num = 14
ThreadA num = 15
ThreadA num = 16
ThreadA num = 17
ThreadB num = 12
ThreadB num = 18
ThreadB num = 19
```

# ThreadLocal 使用示例

上面的两个例子无论是否同步，是否线程安全，他们有一个事实就是都共享了数据，有时候需要各个线程都使用自己的供独立的"全局变量"，这个时候就需要使用 ThreadLocal 了，每个使用 ThreadLcoal 变量的线程都会初始化一个完全独立的实例副本，对其他线程都是不可见的。

ThreaA 方法：

```java
public class ThreadA extends Thread {
    private ThreadLocal<LocalParam> threadLocal;

    public ThreadA(ThreadLocal<LocalParam> threadLocal) {
        this.threadLocal = threadLocal;
    }

    @Override
    public void run() {
        super.run();
        LocalParam localParam = threadLocal.get();
        for (int i = 0; i < 10; i++) {
            localParam.setNum(localParam.getNum() + 1);
            System.out.println("ThreadA num = " + localParam.getNum());
        }
    }
}
```

ThreadB 方法：

```java
public class ThreadB extends Thread {
    private ThreadLocal<LocalParam> threadLocal;

    public ThreadB(ThreadLocal<LocalParam> threadLocal) {
        this.threadLocal = threadLocal;
    }

    @Override
    public void run() {
        super.run();
        LocalParam localParam = threadLocal.get();
        for (int i = 0; i < 10; i++) {
            localParam.setNum(localParam.getNum() + 1);
            System.out.println("ThreadB num = " + localParam.getNum());
        }
    }
}
```

Main 方法：

```java
public class Main {
    private static ThreadLocal<LocalParam> threadLocal = new ThreadLocal<>();
    public static void main(String[] args) {
        threadLocal.set(new LocalParam());
        ThreadA threadA = new ThreadA(threadLocal);
        ThreadB threadB = new ThreadB(threadLocal);
        threadA.start();
        threadB.start();
    }
}
```

运行结果如下：

```text
Exception in thread "Thread-0" java.lang.NullPointerException at com.lupw.spring.thread.local.ThreadB.run(ThreadB.java:16)
Exception in thread "Thread-1" java.lang.NullPointerException at com.lupw.spring.thread.local.ThreadA.run(ThreadA.java:16)
```

为什么呢，我们看源码，ThreadLcal 的 set 方法获取的是当前运行线程，并将当前线程的作为 map 的 key 来保存这个变量的，这里 mian 方法的里面设置的 loaclParam 只是保存了 mian 线程的那个变量，和线程 A 以及线程 B 没有任何关系，所以当我们再线程里面调用 threadLocal.get() 方法时当然会出现空指针异常。

```java
public void set(T value) {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null)
        map.set(this, value);
    else
        createMap(t, value);
}

public T get() {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null) {
        ThreadLocalMap.Entry e = map.getEntry(this);
        if (e != null) {
            @SuppressWarnings("unchecked")
            T result = (T) e.value;
            return result;
        }
    }
    return setInitialValue();
}
```

每一个线程应该都需要使用 ThreadLocal 来设置一个 LocalParam 的值，这里为了简单演示，直接在线程里面去设置 LocalParam 的值。

ThradA 类修改后的代码：

```java
public class ThreadA extends Thread {
    private ThreadLocal<LocalParam> threadLocal;

    public ThreadA(ThreadLocal<LocalParam> threadLocal) {
        this.threadLocal = threadLocal;
    }

    @Override
    public void run() {
        super.run();
        threadLocal.set(new LocalParam());
        LocalParam localParam = threadLocal.get();
        for (int i = 0; i < 10; i++) {
            localParam.setNum(localParam.getNum() + 1);
            System.out.println("ThreadA num = " + localParam.getNum());
        }
    }
}
```

ThreadB 类修改后的代码：

```java
public class ThreadB extends Thread {
    private ThreadLocal<LocalParam> threadLocal;

    public ThreadB(ThreadLocal<LocalParam> threadLocal) {
        this.threadLocal = threadLocal;
    }

    @Override
    public void run() {
        super.run();
        threadLocal.set(new LocalParam());
        LocalParam localParam = threadLocal.get();
        for (int i = 0; i < 10; i++) {
            localParam.setNum(localParam.getNum() + 1);
            System.out.println("ThreadB num = " + localParam.getNum());
        }
    }
}
```

运行结果输出如下，每个线程取用的都是自己的数据进行输出：

```text
Connected to the target VM, address: '127.0.0.1:54492', transport: 'socket'
Disconnected from the target VM, address: '127.0.0.1:54492', transport: 'socket'
ThreadA num = 1
ThreadB num = 1
ThreadA num = 2
ThreadB num = 2
ThreadA num = 3
ThreadB num = 3
ThreadA num = 4
ThreadB num = 4
ThreadA num = 5
ThreadA num = 6
ThreadA num = 7
ThreadA num = 8
ThreadA num = 9
ThreadA num = 10
ThreadB num = 5
ThreadB num = 6
ThreadB num = 7
ThreadB num = 8
ThreadB num = 9
ThreadB num = 10
```