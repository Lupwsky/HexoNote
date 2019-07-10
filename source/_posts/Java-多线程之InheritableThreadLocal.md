---
title: Java-多线程之InheritableThreadLocal
date: 2019-06-19 13:32:06
categories: Java
---

# InheritableThreadLocal

同一个 ThreadLocal 变量在父线程中被设置后, 在子线程中使用 ThreadLocal.get.() 方法是无法获取到得的, 因为两者是不同的线程, 如下例所示:

```java
public static ThreadLocal<String> threadLocal = new ThreadLocal<String>();
public static void main(String[] args) {
    // 在主线程中设置值
    threadLocal.set("A");

    Thread thread = new Thread(new Runbnable() {
        public void run() {
            log.info("value = {}", threadLocal.get());
        }
    });
    thread.start();
}
```

线程中获取的本地变量为 null

<!-- more -->

如果需要在类似这样的场景下使用 ThreadLocal, 可以使用 InheritableThreadLocal 类, InheritableThreadLocal 继承自 ThreadLocal, 允许子线程访问父线程中设置的本地变量, 使用方法和 ThreadLocal 一样, 使用实例如下:

```java
@Slf4j
public class InheritableThreadLocalMain {

    private static InheritableThreadLocal<String> inheritableThreadLocal = new InheritableThreadLocal<>();

    public static void main(String[] args) {
        inheritableThreadLocal.set("A");

        // 在创建线程的时候, 将 InheritableThreadLocal 中的值拷贝一份到线程中
        Thread thread = new Thread(() -> {
            log.info("value = {}", inheritableThreadLocal.get());
            // 在子线程中将值设置为 B
            inheritableThreadLocal.set("B");
        });

        // 先 start 再 join, 让 main 线程等待子线程执行完成
        thread.start();
        try {
            // 等待线程执行完成
            thread.join();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
  
        // 子线程中虽然能够获取到 main 线程中设置的值
        // 但是在子线程中设置 InheritableThreadLocal 的值并不会影响 main 线程中 InheritableThreadLocal 的值
        log.info("value = {}", inheritableThreadLocal.get());
    }
}
```

输出的结果如下:

```text
[2019-06-19 20:33:31][INFO ][com.lupw.guava.thread.InheritableThreadLocalMain] - value = A
[2019-06-19 20:33:31][INFO ][com.lupw.guava.thread.InheritableThreadLocalMain] - value = C
```

在项目中使用 InheritableThreadLocal 类比较多的一种情况是在输出日志的时候获取用于链路跟踪的 traceId, 方便日志排查, 在没有使用 InheritableThreadLocal 前使用的是在线程的构造方法中传入这个 traceId, 后来了解了 InheritableThreadLocal 类后就直接使用 InheritableThreadLocal 类保存 traceId, 相对于自定义一个类去继承 Thread 类并在类中定义一个变量保存 traceId 的值就方便很多了

<div class="note default"><p> InheritableThreadLocal 和 ThreadLocal 不同的是在创建线程的时候, 会将当前线程中 inheritableThreadLocals 变量中的值拷贝一份到被创建线程的 inheritableThreadLocals 变量中 </p></div>