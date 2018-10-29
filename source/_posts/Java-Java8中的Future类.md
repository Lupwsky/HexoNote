---
title: Java-Java8中的Future类
date: 2018-10-29 22:20:19
categories:  Java
---

创建线程的 2 种方式, 一种是直接继承 Thread, 另外一种就是实现 Runnable 接口, 这两种方式在执行完任务之后无法获取执行结果, 从 Java 1.5 开始，就提供了 Callable 和 Future 两个类, 通过它们可以在任务执行完毕之后得到任务执行结果, Future 可以监视目标线程调用 call 的情况, 当你调用 Future 的 get 方法以获获取结果时, 当前线程就开始阻塞, 直到 call 方法结束返回结果

# Future 的使用示例

Future 的一个优点是比底层的 Thread 更加容易使用, 使用 Future, 只需要将耗时的操作封装到 Callable, 再提交到 ExecutorService 就可以了, Java 8 之前的 Future 使用, 使用示例如下:

```java
public class FutureMain {

    public static class DelayTask implements Callable<String> {

        @Override
        public String call() throws Exception {
            Thread.sleep(1000);
            return Thread.currentThread().getName();
        }
    }

    public static void main(String[] args) {
        List<Future<String>> futureList = new ArrayList<>();

        // public static ExecutorService newCachedThreadPool() {
        //    return new ThreadPoolExecutor(0,                 // 0 个核心线程
        //                                  Integer.MAX_VALUE, // 2147483647 个非核心线程
        //                                  60L,               // 非核心线程空闲超过 60s 销毁
        //                                  TimeUnit.SECONDS,
        //                                  new SynchronousQueue<Runnable>());
        // }

        DateTime startTime = DateTime.now();
        ExecutorService executorService = Executors.newCachedThreadPool();
        for (int i = 0; i < 1000; i++) {
            futureList.add(executorService.submit(new DelayTask()));
        }

        futureList.forEach(future -> {
            try {
                // get 方法会一直阻塞, future 提供了一个 get(int time, TimeUnit) 方法, 在阻塞指定时间后, 如果没有获取到结果, 就退出
                String threadName = future.get();
                log.info("[Future 测试] threadName = {}", threadName);
            } catch (InterruptedException | ExecutionException e) {
                e.printStackTrace();
            }
        });
        DateTime endTime = DateTime.now();
        log.info("[Future 测试] 总耗时 = {}ms", endTime.getMillis() - startTime.getMillis());

        if (!executorService.isShutdown()) {
            executorService.shutdown();
        }
    }
}
```

<!-- more -->

输出结果如下:

```text
22:50:46.557 [main] INFO com.thread.excutor.FutureMain - [Future 测试] threadName = pool-1-thread-1
22:50:46.560 [main] INFO com.thread.excutor.FutureMain - [Future 测试] threadName = pool-1-thread-2
...
22:50:46.644 [main] INFO com.thread.excutor.FutureMain - [Future 测试] threadName = pool-1-thread-999
22:50:46.644 [main] INFO com.thread.excutor.FutureMain - [Future 测试] threadName = pool-1-thread-1000
22:50:46.644 [main] INFO com.thread.excutor.FutureMain - [Future 测试] 总耗时 = 1166ms
```

这里是一些参考的资料: [Furure和FutureTask的区别](https://blog.csdn.net/bboyfeiyu/article/details/24851847), [Java中的Runnable、Callable、Future、FutureTask的区别与示例](http://www.voidcn.com/article/p-nvubkexm-bac.html)

# Java 8 中的 Furure
