---
title: Java-Guava工具类之事件总线
date: 2019-01-26 14:04:53
categories: Java
---

之前一直有听说过谷歌的 Guava, 但是在项目中也仅是非常简单使用过其中的缓存功能 , 称着这段时间项目不是很忙, 抓紧时间系统的了解下 (主要参考 [官方的文档](https://github.com/google/guava) 和 [Guava官方教程(中文版)](https://willnewii.gitbooks.io/google-guava/content/index.html)), 这里记录下自己的学习笔记

# EventBus

Java 的进程内事件分发都是通过发布者和订阅者之间的显式注册实现的, Guava 的 EventBus 可以实现进程内事件分发, 使组件间有了更好的解耦, EventBus 不适用于进程间通信, Guava 为我们提供了同步事件 EventBus 和异步实现 AsyncEventBus 两个事件总线

# 订阅事件

订阅一个事件很简单, 在方法上面添加一个 @Subscribe 和保证只有一个输入参数的方法就可

```java
@Slf4j
public class EventBusSubscriber {

    @Subscribe
    public void eventBusListener(String event) {
        // 打印线程名和事件的值
        log.info("[eventBusListener] thread = {}, value = {}", 
            Thread.currentThread().getName(), event);
    }
}
```

<!-- more -->

注: Guava 发布的事件默认不会处理线程安全的, 可以使用注解标注 @AllowConcurrentEvents 来保证其线程安全

# 发布事件

通过 EventBus.post() 来发布事件, EventBus 提供了同步和异步两种方式来发布事件

## 同步发布

同步发布事件, 会阻塞当前线程, 等待所有事件处理完成

```java
EventBus eventBus = new EventBus();
eventBus.register(new EventBusSubscriber());
eventBus.post("Hello, Guava EventBus");
```

输出结果如下:

```text
# 事件处理和发布在同一个线程里面
[eventBusListener] thread = main, value = Hello, Guava EventBus
```

## 异步发布

同步发布事件, 不会阻塞线程

```java
AsyncEventBus asyncEventBus = new AsyncEventBus(new SimpleAsyncTaskExecutor());
asyncEventBus.register(new EventBusSubscriber());
asyncEventBus.post("Hello, Guava EventBus");
```

输出结果如下:

```text
# 事件处理和发布不在同一个线程里面
[eventBusListener] SimpleAsyncTaskExecutor-1, value = Hello, Guava EventBus
```

# 死亡事件

如果 EventBus 发送的消息都不是订阅者关心的称之为 DeadEvent, 即没没有任何一个订阅者订阅这个事件, 实现一个 DeadEvent 只需要保证方法有一个参数且参数类型为 DeadEvent 即可

```java
@Slf4j
public class EventBusSubscriber {

    @Subscribe
    public void eventBusListener(String event) {
        log.info("[eventBusListener] thread = {}, value = {}",
            Thread.currentThread().getName(), event);
    }


    @Subscribe
    public void deadEventBusListener(DeadEvent event) {
        log.info("[deadEventBusListener] thread = {}, value = {}",
            Thread.currentThread().getName(), event);
    }
}
```

测试死亡事件:

```java
EventBus eventBus = new EventBus();
eventBus.register(new EventBusSubscriber());
eventBus.post("DeadEvent");
```

输出结果如下:

```java
[eventBusListener] thread = main, value = DeadEvent
```