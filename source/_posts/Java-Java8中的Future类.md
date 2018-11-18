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

在 Java 8 中新设计了一个 CompletableFuture 类, 极大的扩展了 Future 类,  Future 类提供了异步执行任务并获取返回结果的能力, 但是对于结果的获取只能使用 get 方法, 会造成阻塞, 而且 CompletableFuture 类新支持了观察者设计模式, 当计算结果完成时及时通知监听者, 能有效的避免阻塞, 为什么会新设计这样一个 Future 类呢? 虽然 Future 能狗很方便完成异步任务并获取到返回值, 但是任然有很多的局限性, 新的 CompletableFuture 类能够解决这些问题, Future 类的局限性情况如下:

* 两个异步任务计算相互独立, 但是第二个又依赖第一个计算的结果
* 等待 Future 集合中的所有任务全部完成
* 仅等待 Future 集合任中最快的任务执行完成
* Future 完成时发送一个通知, 并能使用 Future 的结果进行下一步的操作

## 创建 CompletableFuture 对象

CompletableFuture 提供了四个静态方法用于获取 CompletableFuture 实例, 如下:

```java
public static CompletableFuture<Void> runAsync(Runnable runnable);
public static CompletableFuture<Void> runAsync(Runnable runnable, Executor executor);
public static <U> CompletableFuture<U> supplyAsync(Supplier<U> supplier);
public static <U> CompletableFuture<U> supplyAsync(Supplier<U> supplier, Executor executor)
```

runAsync 和 supplyAsync 方法区别是一个接收 Runnable 对象一个接收 Calleable 对象, 一个没有返回值, 一个有返回值, 两个方法都有一个重载的方法, 如果不传递 Executor 或者传递的是一个 null 值, 则默认使用 Fork/Join 框架的 Executor, 以 supplyAsync 为例, 可以从其源码清楚的了解到这一情况:

```java
private static final boolean useCommonPool = (ForkJoinPool.getCommonPoolParallelism() > 1);
private static final Executor asyncPool = useCommonPool ? ForkJoinPool.commonPool() : new ThreadPerTaskExecutor();
public static <U> CompletableFuture<U> supplyAsync(Supplier<U> supplier) {
    return asyncSupplyStage(asyncPool, supplier);
}
```

## getNow 方法和 join 方法

CompletableFuture 可以像 Future 一样使用 get 阻塞的方式获取结果, 使用方法和 Future 一样, 这里不举例说明了, CompletableFuture 还添加了两个方法 getNow(T valueIfAbsent) 和 join() 方法, getNow 方法如果计算的完成则返回结果, 否则返回默认值, getNow 方法的使用示例如下:

```java
CompletableFuture<Integer> future = CompletableFuture.supplyAsync(() -> {
    try {
        // 模拟计算需要 5s 才能完成
        Thread.sleep(5000);
    } catch (InterruptedException e) {
        e.printStackTrace();
    }
    return 100;
});

try {
    Thread.sleep(1000);
} catch (InterruptedException e) {
    e.printStackTrace();
}

// 1s 后获取计算结果, 此时计算还没有完成, 返回默认值 200
int result = future.getNow(200);
log.info("[MAIN] result = {}", result);
```

输出结果如下:

```text
[main] INFO com.thread.excutor.CompletableFutureMain - [MAIN] result = 200
```

join 方法使用和 get 方法是一样的效果, 只是 join 方法不需要和 get 方法抛出受检测的 ExecutionException 异常和 InterruptedException 异常, 即编译器也不会强制你加上 tray-catch 语句, 同时 join 也没有和 get 一样有一个重载方法可以在获取结果时指定超时时间, 使用示例如下:

```java
CompletableFuture<Integer> future = CompletableFuture.supplyAsync(() -> 100);

// 不需要加 tray-catch 语句捕获异常
int result = future.join();
log.info("[MAIN] result = {}", result);
```

## thenApply 方法

thenApply 方法用于将多个 CompletableFuture 的在获取到结果重新进行计算或者组合, 并将返回结果转换成其他的类型 (可参考 Stream API 中的 map 方法理解), 一共有三个重载的方法, 如下:

```java
public <U> CompletableFuture<U> thenApply(Function<? super T,? extends U> fn);
public <U> CompletableFuture<U> thenApplyAsync(Function<? super T,? extends U> fn);
public <U> CompletableFuture<U> thenApplyAsync(Function<? super T,? extends U> fn, Executor executor);
```

里面有两个方法后面带有 Async 的后缀, 着两个方法和 thenApply 不同的是重新计算结果的时候, 多个 CompletableFuture 执行 thenApply 都是在同一个线程里面执行的, 而 thenApplyAsync 会在不同的线程里面执行 (也有可能在同一个线程里面执行), 后面介绍的方法后缀带有 Async 都是类似的, thenApply 方法的使用实例如下:

```java
public static void main(String[] args) {
    log.info("[MAIN] threadName = {}, TAG-1", Thread.currentThread().getName());
    CompletableFuture<Integer> future = CompletableFuture.supplyAsync(() -> {
        try {
            log.info("[MAIN] threadName = {}, FUTURE-1", Thread.currentThread().getName());
            Thread.sleep(2000);
            log.info("[MAIN] threadName = {}, FUTURE-1 2s 过去了...", Thread.currentThread().getName());
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        return 100;
    });

    log.info("[MAIN] threadName = {}, TAG-2", Thread.currentThread().getName());
    String str = future.thenApply((result) -> {
        try {
            log.info("[MAIN] threadName = {}, FUTURE-2", Thread.currentThread().getName());
            Thread.sleep(1000);
            log.info("[MAIN] threadName = {}, FUTURE-2 1s 过去了...", Thread.currentThread().getName());
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        return String.valueOf(result) + "1";
    }).join();

    log.info("[MAIN] threadName = {}, TAG-3", Thread.currentThread().getName());
    log.info("[MAIN] threadName = {}, result = {}", Thread.currentThread().getName(), str);
}
```

输出结果如下:

```text
16:47:18.762 [main] INFO com.thread.excutor.CompletableFutureMain - [MAIN] threadName = main, TAG-1
16:47:18.825 [main] INFO com.thread.excutor.CompletableFutureMain - [MAIN] threadName = main, TAG-2
16:47:18.825 [ForkJoinPool.commonPool-worker-1] INFO com.thread.excutor.CompletableFutureMain - [MAIN] threadName = ForkJoinPool.commonPool-worker-1, FUTURE-1
16:47:20.833 [ForkJoinPool.commonPool-worker-1] INFO com.thread.excutor.CompletableFutureMain - [MAIN] threadName = ForkJoinPool.commonPool-worker-1, FUTURE-1 2s 过去了...
16:47:20.833 [ForkJoinPool.commonPool-worker-1] INFO com.thread.excutor.CompletableFutureMain - [MAIN] threadName = ForkJoinPool.commonPool-worker-1, FUTURE-2
16:47:21.849 [ForkJoinPool.commonPool-worker-1] INFO com.thread.excutor.CompletableFutureMain - [MAIN] threadName = ForkJoinPool.commonPool-worker-1, FUTURE-2 1s 过去了...
16:47:21.849 [main] INFO com.thread.excutor.CompletableFutureMain - [MAIN] threadName = main, TAG-3
16:47:21.849 [main] INFO com.thread.excutor.CompletableFutureMain - [MAIN] threadName = main, result = 1001
```

从输出的日志可以的时间可以看到, thenApply 是按照顺序在执行的, 两个 Future 执行完一共花了 3s 时间

## thenAccept 和 thenRun 方法

thenAccept 是在获取到结果后执行相应的操作, 这个过程是非阻塞的, 也就是前面提到的  CompletableFuture 类支持的观察者设计模式, 当计算结果完成时及时通知监听者, 执行相关的操作, thenRun 方法和 thenAccept 方法一样, 只不过 thenRun 方法接收一个 Runnable, 不带有返回值, 相关的方法如下:

```java
public CompletableFuture<Void> thenAccept(Consumer<? super T> action);
public CompletableFuture<Void> thenAcceptAsync(Consumer<? super T> action);
public CompletableFuture<Void> thenAcceptAsync(Consumer<? super T> action, Executor executor);

public CompletableFuture<Void> thenRun(Runnable action);
public CompletableFuture<Void> thenRunAsync(Runnable action);
public CompletableFuture<Void> thenRunAsync(Runnable action, Executor executor);
```

thenAccept 的使用示例如下:

```java
public static void main(String[] args) {
    CompletableFuture<Integer> future = CompletableFuture.supplyAsync(() -> {
        try {
            Thread.sleep(2000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        return 100;
    });

    future.thenAccept((result) -> {
        try {
            log.info("[MAIN] threadName = {}", Thread.currentThread().getName());
            Thread.sleep(1000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    });
    log.info("[MAIN] 不管 thenAccept 里面的方法是否执行完成, 我先执行");

    try {
        Thread.sleep(4000);
    } catch (InterruptedException e) {
        e.printStackTrace();
    }
    log.info("[MAIN] 程序结束");
}
```

输出结果如下:

```text
18:05:04.220 [main] INFO com.thread.excutor.CompletableFutureMain - [MAIN] 不管 thenAccept 里面的方法是否执行完成, 我先执行
18:05:06.221 [ForkJoinPool.commonPool-worker-1] INFO com.thread.excutor.CompletableFutureMain - [MAIN] threadName = ForkJoinPool.commonPool-worker-1
18:05:08.221 [main] INFO com.thread.excutor.CompletableFutureMain - [MAIN] 程序结束
```

## thenCompose 方法

thenCompose 用来连接两个 CompletableFuture, 尤其适用于上一个 future 的结果被下一个 future 依赖的情况, thenApply 方法也能是实现类似的功能, 但是 thenCompose 和 thenApply 不同的是 thenCompose 最终生成的是一个新的 CompletableFuture, 但是 thenApply 只是转换了泛型中的类型, 并没有产生新的 CompletableFuture, 但是要注意的是这都是一个串行实现, thenCompose 有以下重载方法:

```java
public <U> CompletableFuture<U> thenCompose(Function<? super T, CompletableFuture<U>> fn);
public <U> CompletableFuture<U> thenComposeAsync(Function<? super T, CompletableFuture<U>> fn);
public <U> CompletableFuture<U> thenComposeAsync(Function<? super T, CompletableFuture<U>> fn,Executor executor);
```

使用示例如下:

```java
public static void main(String[] args) {
    // 模拟需要计算 2s 时间得出结果, 立即开始计算
    CompletableFuture<Integer> future1 = CompletableFuture.supplyAsync(() -> {
        try {
            Thread.sleep(2000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        return 100;
    });

    // 2s 后 future1 计算出结果后才会开始计算, 将结果的值参与 future2 的计算
    CompletableFuture<Integer> future2 = future1.thenCompose(future1Result -> CompletableFuture.supplyAsync(() -> {
        try {
            Thread.sleep(3000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        return future1Result * 10;
    }));

    try {
        // 阻塞获取最终的结果, 整个过程需要 5s (因为是串行实现)
        int finalResult = future2.get(10, TimeUnit.SECONDS);
        log.info("[MAIN] finalResult = {}", finalResult);
    } catch (InterruptedException | TimeoutException | ExecutionException e) {
        e.printStackTrace();
    }
}
```

输出结果如下:

```text
19:37:20.260 [main] INFO com.thread.excutor.CompletableFutureMain - [MAIN] finalResult = 1000
```

## thenCombine 方法

thenCombine 也是于连接两个 future, 和 thenCompose 不同的是 thenCombine 是并行实现的, 两个 future 同时开始计算, 最终的结果依赖两个 future 的结果, 而 thenCompose 用于第二个 future 的计算依赖第一个 future 的计算结果的情况, thenCombine 也是会新生成一个 CompletableFuture 对象, 有以下几个重载方法:

```java
public <U,V> CompletableFuture<V> thenCombine(CompletableFuture<? extends U> other, BiFunction<? super T,? super U,? extends V> fn);
public <U,V> CompletableFuture<V> thenCombineAsync(CompletableFuture<? extends U> other, BiFunction<? super T,? super U,? extends V> fn);
public <U,V> CompletableFuture<V> thenCombineAsync(CompletableFuture<? extends U> other, BiFunction<? super T,? super U,? extends V> fn,Executor executor);
```

```java
CompletableFuture<Integer> future1 = CompletableFuture.supplyAsync(() -> {
    try {
        Thread.sleep(2000);
    } catch (InterruptedException e) {
        e.printStackTrace();
    }
    return 100;
});

CompletableFuture<Integer> future2 = future1.thenCombine(CompletableFuture.supplyAsync(() -> {
    try {
        Thread.sleep(3000);
    } catch (InterruptedException e) {
        e.printStackTrace();
    }
    return 200;
}), (future1Result, otherFutureResult) -> {
    log.info("[MAIN] future1Result = {}", future1Result);
    log.info("[MAIN] otherFutureResult = {}", otherFutureResult);
    return future1Result + otherFutureResult;
});

try {
    // 阻塞获取最终的结果, 整个过程需要 3s (并行实现的)
    DateTime startTime = DateTime.now();
    int finalResult = future2.get(10, TimeUnit.SECONDS);
    DateTime endTime = DateTime.now();
    log.info("[MAIN] finalResult = {}, time = {}ms", finalResult, endTime.getMillis() - startTime.getMillis());
} catch (InterruptedException | TimeoutException | ExecutionException e) {
    e.printStackTrace();
}
```

输出结果如下:

```text
19:56:00.635 [ForkJoinPool.commonPool-worker-2] INFO com.thread.excutor.CompletableFutureMain - [MAIN] future1Result = 100
19:56:00.635 [ForkJoinPool.commonPool-worker-2] INFO com.thread.excutor.CompletableFutureMain - [MAIN] otherFutureResult = 200
19:56:00.635 [main] INFO com.thread.excutor.CompletableFutureMain - [MAIN] finalResult = 300, time = 2988ms
```

这里日志输出的总时间小于 3s 是由于时间的计算只计算了 future 调用 get 方法阻塞的那一段时间, 有一点点小误差

## whenComplete 方法

whenComplete 可以在 future 计算结果结束后调用这个方法, 接收一个 BiConsumer 接口类型的参数, 两个参数一个是计算的结果值, 一个是计算时抛出异常类型, 此方法返回的任然是原始的 CompletableFuture 对象, 有以下几个重载方法:

```java
public CompletionStage<T> whenComplete(BiConsumer<? super T, ? super Throwable> action);
public CompletionStage<T> whenCompleteAsync(BiConsumer<? super T, ? super Throwable> action);
public CompletionStage<T> whenCompleteAsync(BiConsumer<? super T, ? super Throwable> action,Executor executor);
```

使用示例如下:

```java
public static void main(String[] args) {
    CompletableFuture<Integer> future1 = CompletableFuture.supplyAsync(() -> {
        try {
            Thread.sleep(2000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        return 100;
    }).whenComplete((result, error) -> log.info("[Main] result = {}, error = {}", result, error.getMessage()));

    try {
        DateTime startTime = DateTime.now();
        int finalResult = future1.get(10, TimeUnit.SECONDS);
        DateTime endTime = DateTime.now();
        log.info("[MAIN] finalResult = {}, time = {}ms", finalResult, endTime.getMillis() - startTime.getMillis());
    } catch (InterruptedException | TimeoutException | ExecutionException e) {
        e.printStackTrace();
    }
}
```

输出结果如下:

```text
java.util.concurrent.ExecutionException: java.lang.NullPointerException
    at java.util.concurrent.CompletableFuture.reportGet(CompletableFuture.java:357)
    at java.util.concurrent.CompletableFuture.get(CompletableFuture.java:1915)
    at com.thread.excutor.CompletableFutureMain.main(CompletableFutureMain.java:26)
Caused by: java.lang.NullPointerException
    at com.thread.excutor.CompletableFutureMain.lambda$main$1(CompletableFutureMain.java:22)
    at java.util.concurrent.CompletableFuture.uniWhenComplete(CompletableFuture.java:760)
    at java.util.concurrent.CompletableFuture$UniWhenComplete.tryFire(CompletableFuture.java:736)
    at java.util.concurrent.CompletableFuture.postComplete(CompletableFuture.java:474)
    at java.util.concurrent.CompletableFuture$AsyncSupply.run$$$capture(CompletableFuture.java:1595)
    at java.util.concurrent.CompletableFuture$AsyncSupply.run(CompletableFuture.java)
    at java.util.concurrent.CompletableFuture$AsyncSupply.exec(CompletableFuture.java:1582)
    at java.util.concurrent.ForkJoinTask.doExec(ForkJoinTask.java:289)
    at java.util.concurrent.ForkJoinPool$WorkQueue.runTask(ForkJoinPool.java:1056)
    at java.util.concurrent.ForkJoinPool.runWorker(ForkJoinPool.java:1692)
    at java.util.concurrent.ForkJoinWorkerThread.run(ForkJoinWorkerThread.java:157)
```

这里出现了一个异常, 日志提示是调用 get 方法出现的异常, 实际上时由于 future 在计算的时候没有出现异常, 在输出日志的时候使用了 error.getMessage() 出现的 (Lamdba 调试时不容易定位问题), 稍微改一下即可

## thenAcceptBoth 方法

thenAcceptBoth 方法的作用和 thenCombine 方法一样用于串行连接两个 future, 有一点区别就是 thenCombine 会有一个返回值, 而 thenAcceptBoth 方法没有, 调用方法和 thenCombine 一样, 这里就不贴出示例代码了 相关的方法定义如下:

```java
public <U> CompletionStage<Void> thenAcceptBoth(CompletionStage<? extends U> other, BiConsumer<? super T, ? super U> action);
public <U> CompletionStage<Void> thenAcceptBothAsync(CompletionStage<? extends U> other, BiConsumer<? super T, ? super U> action);
public <U> CompletionStage<Void> thenAcceptBothAsync(CompletionStage<? extends U> other, BiConsumer<? super T, ? super U> action, Executor executor);
```

## runAfterBoth 方法

runAfterBoth 方法用于并行连接两个 future, 两个 future 相互之间没有结果上的依赖, 并且在两个 future 都计算完成后, 调用其中一个 future 的 get 方法后, 就会执行 Runnable 的 run 方法, 相关方法的定义如下:

```java
public CompletionStage<Void> runAfterBoth(CompletionStage<?> other,Runnable action);
public CompletionStage<Void> runAfterBothAsync(CompletionStage<?> other,Runnable action);
public CompletionStage<Void> runAfterBothAsync(CompletionStage<?> other,Runnable action, Executor executor);
```

使用示例如下:

```java
CompletableFuture<Integer> future1 = CompletableFuture.supplyAsync(() -> {
    try {
        Thread.sleep(1000);
    } catch (InterruptedException e) {
        e.printStackTrace();
    }
    return 100;
});

CompletableFuture<Integer> future2 = CompletableFuture.supplyAsync(() -> {
    try {
        Thread.sleep(5000);
    } catch (InterruptedException e) {
        e.printStackTrace();
    }
    return 200;
});

future1.runAfterBoth(future2, () -> log.info("[MAIN] 5s 后所有计算完成").join;
```

输出结果如下:

```text
[ForkJoinPool.commonPool-worker-2] INFO com.thread.excutor.CompletableFutureMain - [MAIN] 5s 后所有计算完成
```

## runAfterEither 方法

和 runAfterBoth 方法相反, runAfterBoth 方法用于连接两个 future, 但是只要任何一个 future 执行完成就会调用 Runnable 的方法, 使用方法和 runAfterBoth 方法类似, 这里就不贴出示例代码了, 相关方法的定义如下:

```java
public CompletionStage<Void> runAfterEither(CompletionStage<?> other, Runnable action);
public CompletionStage<Void> runAfterEitherAsync(CompletionStage<?> other, Runnable action);
public CompletionStage<Void> runAfterEitherAsync(CompletionStage<?> other, Runnable action, Executor executor);
```

## applyToEither 方法

用于连接两个 future, 选择计算结果最快的一个 future 返回, 并对 CompletableFuture 的泛型类型进行转换, 使用方法和 thenApply 类似, 这里就不贴出测试代码了, 相关方法的定义如下:

```java
public <U> CompletionStage<U> applyToEither(CompletionStage<? extends T> other, Function<? super T, U> fn);
public <U> CompletionStage<U> applyToEitherAsync(CompletionStage<? extends T> other, Function<? super T, U> fn);
public <U> CompletionStage<U> applyToEitherAsync(CompletionStage<? extends T> other, Function<? super T, U> fn, Executor executor);
```

## acceptEither 方法

用于连接连接两个 future, 选择计算结果最快的一个 future 返回, 和 applyToEither 方法不同的是 acceptEither 方法没有返回值, 不能进行诸如对 CompletableFuture 的泛型类型进行转换的操作, 使用方法和 thenAccept 类似, 这里就不贴出测试代码了, 相关方法的定义如下:

```java
public CompletionStage<Void> acceptEither(CompletionStage<? extends T> other, Consumer<? super T> action);
public CompletionStage<Void> acceptEitherAsync(CompletionStage<? extends T> other, Consumer<? super T> action);
public CompletionStage<Void> acceptEitherAsync(CompletionStage<? extends T> other, Consumer<? super T> action, Executor executor);
```

## exceptionally 方法

exceptionally 用于 future 捕捉在计算的时候出现的异常, 在调用 get 方法如果计算是出现异常时就会被调用, 使用示例如下:

```java
CompletableFuture<Integer> future1 = CompletableFuture.supplyAsync(() -> {
    if (true) {
        throw new RuntimeException();
    }
    return 100;
}).whenComplete((result, error) -> log.info("[Main] result = {}, error = {}", result, error.getMessage())).exceptionally(e -> 500);
```

输出结果如下:

```text
[main] INFO com.thread.excutor.CompletableFutureMain - [Main] result = null, error = java.lang.RuntimeException
```

可以当出现异常导致计算结束时 whenComplete 也会被调用, 即使没有调用 get 方法, 但是 exceptionally 则不会调用, 加上 get 方法的调用, 示例代码如下:

```java
CompletableFuture<Integer> future1 = CompletableFuture.supplyAsync(() -> {
    if (true) {
        throw new RuntimeException();
    }
    return 100;
}).whenComplete((result, error) -> log.info("[Main] result = {}, error = {}", result, error.getMessage())).exceptionally(e -> 500);

try {
    DateTime startTime = DateTime.now();
    int finalResult = future1.get(10, TimeUnit.SECONDS);
    DateTime endTime = DateTime.now();
    log.info("[MAIN] finalResult = {}, time = {}ms", finalResult, endTime.getMillis() - startTime.getMillis());
} catch (InterruptedException | TimeoutException | ExecutionException e) {
    e.printStackTrace();
}
```

输出结果如下:

```text
[main] INFO com.thread.excutor.CompletableFutureMain - [Main] result = null, error = java.lang.RuntimeException
[main] INFO com.thread.excutor.CompletableFutureMain - [MAIN] finalResult = 500, time = 31ms
```

可以看到这个时候 whenComplete 和 exceptionally 方法都被调用了, 并且 exceptionally 方法返回的值时直接影响到了 whenComplete 获取的值, whenComplete 获取到的是 exceptionally  最终返回的值

## completeExceptionally 方法

completeExceptionally 方法也是 future 处理异常的一个方法, 和 exceptionally 方法不同的是, exceptionally 捕捉的是在计算时出现的异常, 然后在调用 get 方法时会调用 exceptionally 方法对异常处理, 而 completeExceptionally 是在任意地方可以调用, 并在调用 get 方法时抛出这个异常, 无论 future 是否计算完成都会抛出这个异常, completeExceptionally 异常处理和 exceptionally 处理可以同时使用, 他们对应的使用场景时不同的, 使用示例如下:

```java
CompletableFuture<Integer> future1 = CompletableFuture.supplyAsync(() -> {
    try {
        Thread.sleep(1000);
    } catch (InterruptedException e) {
        e.printStackTrace();
    }
    return 100;
}).whenComplete((result, error) -> log.info("[Main] result = {}, error = {}", result, error.getMessage()));

// 等待 futrue 计算完成后抛出异常
try {
    Thread.sleep(2000);
} catch (InterruptedException e) {
    e.printStackTrace();
}
future1.completeExceptionally(new UnsupportedOperationException());

try {
    DateTime startTime = DateTime.now();
    // 虽然 future 已经计算完成, 任然抛出 UnsupportedOperationException 异常
    int finalResult = future1.get(10, TimeUnit.SECONDS);
    DateTime endTime = DateTime.now();
    log.info("[MAIN] finalResult = {}, time = {}ms", finalResult, endTime.getMillis() - startTime.getMillis());
} catch (InterruptedException | TimeoutException | ExecutionException e) {
    e.printStackTrace();
}
```

## handle 方法

handle 方法和 whenComplete 的效果是一样的, 但是和 whenComplete 稍微有一点不同, handle 有一个返回值, 同时和 whenComplete 一样第二个参数时出现的异常, 可以在这里对异常进行一些操作, 例如抛出异常让 exceptionally 方法可以捕捉和处理, 或者出现异常返回一个默认的值, 使用方法和 whenComplete 类似这里就不贴出使用示例代码了, 相关的方法如下:

```java
public <U> CompletionStage<U> handle(BiFunction<? super T, Throwable, ? extends U> fn);
public <U> CompletionStage<U> handleAsync(BiFunction<? super T, Throwable, ? extends U> fn);
public <U> CompletionStage<U> handleAsync(BiFunction<? super T, Throwable, ? extends U> fn,Executor executor);
```

# 参考资料

* Java 8 实战 (书籍)
* JDK 8 CompletableFuture 类源码