---
title: Spring-SpringRetry重试实现机制
date: 2019-03-11 22:44:41
categories: Spring
---

之前项目中有一个同步数据的定时任务, 在每天的 01:00 开始执行, 需要调用其他公司提供的 HTTP 接口来同步数据, 由于使用的 HTTP 获取数据, 为了防止网络波动造成有些请求失败, 需要添加重试机制, 要求为失败的话 2 秒后再次发起请求, 最多请求三次, 于是我写下如下逻辑代码, 这里贴出伪代码

```java
// 该方法在子线程里面执行的
private boolean failureRetry(String traceId) {
    int tryCount = 0;
    for (int i = 0; i < 3; i++) {
        tryCount = i + 1;
        log.info("[同步服务小结重试] 重新发起请求, traceId = {}, 此次同步重试次数 = {}", traceId, tryCount);
        try {
            Thread.sleep(2000);
        } catch (InterruptedException e) {
            log.info("[同步服务小结重试] 线程异常, traceId = {}, error = {}", traceId, e);
            return false;
        }

        // 模拟逻辑处理的方法, 获取请求返回码
        int code = getData();
        if (code == 200) {
            // 成功的处理逻辑
            return true;
        } else {
            // 失败的处理逻辑
            log.info("[同步服务小结重试] 失败, traceId = {}, 此次同步重试次数 = {}", traceId, tryCount);
        }
    }
    return false;
}
```

<!-- more -->

上面的逻辑实现一个简单的重试机制是可以的, 可以满足项目的需求, 在网上了解重试机制的开源项目后, 发现了 Guava 的 retrying 工具和 spring-retry 项目, 可以更加优雅的实现重试机制, 而且功能更加强大, 因此花些时间学习下, 这篇笔记主要记录 spring-retry 的基础使用方法

# spring-retry 项目

访问 Github 上的项目, [spring-retry](https://github.com/spring-projects/spring-retry), 看看官方对 spring-retry 的介绍:

`This project provides declarative retry support for Spring applications. It is used in Spring Batch, Spring Integration, Spring for Apache Hadoop (amongst others).`
`该项目为 Spring 应用程序提供声明性重试支持。它用于Spring Batch, Spring Integration, Spring for Apache Hadoop (以及其他)`

spring-retry 是从 spring-batch 中分离出来的一个项目, 主要实现了重试和熔断的功能

# 基础使用

一个基本的重试机制分成了三点, 重试的基础处理逻辑, 最终失败的处理逻辑和重试策略, spring-retry 在实现一个重试功能时也包含了这三点要素, 写一个简单的例子如何使用的, 使用方法写在注释里面:

```java
// 创建 RetryTemplate 模板用于组装基础处理逻辑, 兜底操作 (最终失败的处理逻辑) 和重试策略并执行重试任务, RetryTemplate 实现了 RetryOperations 接口
RetryTemplate retryTemplate = new RetryTemplate();

// 创建一个简单的重试策略并添加到 retryTemplate 中, 
// TimeoutRetryPolicy 重试策略是在指定的时间后再次重试一次, 查看源码可以知道默认为 1 秒后, 里面还有有个 isAlive 方法在执行的时候可以判断是否已经超时过期
TimeoutRetryPolicy timeoutRetryPolicy = new TimeoutRetryPolicy();
timeoutRetryPolicy.setTimeout(2000);
retryTemplate.setRetryPolicy(timeoutRetryPolicy);

// 添加基础回调和最终失败处理回调并执行
// 基础回调需要实现 RetryCallback 接口并实现 doWithRetry 方法, 重试的基础处理逻辑在这个方法里面实现, 泛型 T, E 分别表示返回的数据类型和满足重试机制出现的异常类型, 不是此类型的异常不会重试
// 兜底操作需要实现 RecoveryCallback 接口并实现 recover 方法, 最终失败的处理逻辑在这个方法里面实现, 泛型 T 表示返回的数据类型, 一般和 RetryCallback 的保持一致
String result = null;
try {
    result = retryTemplate.execute(new RetryCallback<String, Exception>() {
        @Override
        public String doWithRetry(RetryContext context) throws Exception {
            throw new RetryException("测试");
        }
    }, new RecoveryCallback<String>() {
        @Override
        public String recover(RetryContext context) throws Exception {
            return "失败";
        }
    });
} catch (Exception e) {
    e.printStackTrace();
}
log.info("2s 后 result = {}", result);
```

运行项目, 输出的结果如下:

```text
2s 后 result = 失败
```

# RecoveryCallback

使用 spring-retry 的重试机制时, 基础处理逻辑的回调需要实现 RecoveryCallback 类里面的 doWithRetry 方法, 该方法的定义如下:

```java
public interface RetryCallback<T, E extends Throwable> {
	T doWithRetry(RetryContext context) throws E;
}
```

该方法抛出一个泛型 E 异常, 当执行 doWithRetry 方法出现这个异常的时候就会进行重试, doWithRetry 方法包含一个RetryContext 类型的参数, 即重试上下文, 她保存了在重试的时候的一些状态, 如尝试用的重试的次数和上一次是因为什么类型的异常引起的重试, 这里面包含了异常的栈信息, 下面的例子输出重试的次数和最后一次发生的异常:

```java
RetryTemplate retryTemplate = new RetryTemplate();

// 简单重试策略, 重试到指定的次数, 这也是默认的策略
// 这里因为没有配置回退策略, 这 10 次的重试 (包含第一的调用) 都是等上一次执行完成后就直接执行了
SimpleRetryPolicy simpleRetryPolicy = new SimpleRetryPolicy();
simpleRetryPolicy.setMaxAttempts(10);
retryTemplate.setRetryPolicy(simpleRetryPolicy);

String result = null;
try {
    result = retryTemplate.execute(new RetryCallback<String, Exception>() {
        @Override
        public String doWithRetry(RetryContext context) throws Exception {
            log.info("doWithRetry retry count = {}", context.getRetryCount());
            throw new RetryException("测试");
        }
    }, new RecoveryCallback<String>() {
        @Override
        public String recover(RetryContext context) throws Exception {
            Throwable throwable = context.getLastThrowable();
            log.info("error msg = {}", throwable.getMessage());
            log.info("recover retry count = {}", context.getRetryCount());
            return "失败";
        }
    });
} catch (Exception e) {
    e.printStackTrace();
}
```

启动项目, 输出结果如下:

```text
doWithRetry retry count = 0
doWithRetry retry count = 1
doWithRetry retry count = 2
doWithRetry retry count = 3
doWithRetry retry count = 4
doWithRetry retry count = 5
doWithRetry retry count = 6
doWithRetry retry count = 7
doWithRetry retry count = 8
doWithRetry retry count = 9
error msg = 测试
recover retry count = 10
```

# RecoveryCallback

在到达指定重试次数如果还没有执行成功, 需要有一个兜底的操作, 兜底操作的回调需要实现 RecoveryCallback 类的 recover 方法, 定义如下:

```java
public interface RecoveryCallback<T> {
	T recover(RetryContext context) throws Exception;
}
```

recover 方法也带有重试上下文参数, 另外如果执行出现异常, 可以抛出异常让调用者处理, 或者自己处理掉异常

## 注解方式实现

spring-retry 支持使用注解方法实现, 首先需要在启动类上添加 `@EnableRetry` 注解, 在需要重试的方法上添加 `@Retryable` 注解, @Retryable(RetryException.class) 注解表示调用这个方法, 如果出现 RetryException 异常就重试, `默认最多重试 3 次, 重试间隔为 1 秒` 示例如下:

```java
@Retryable(RetryException.class)
public void doSomething() {
    log.info("[开始执行任务1]");
    throw new RetryException("抛出异常");
}


@Retryable(RetryException.class)
public void doSomething2() {
    log.info("[开始执行任务2]");
    throw new RetryException("抛出异常");
}
```

接着在某个方法上添加 `@Recover`, 表示这是一个兜底方法, 如果最后经过重试的次数还是没有成功就会执行这个方法, 注意在用一个类中有多个 `@Recover` 注解的方法, 只会执行第一个

```java
@Recover
public void doSomethingIfUnsuccessful1() {
    log.info("[任务执行失败1]");
}

@Recover
public void doSomethingIfUnsuccessful2() {
    log.info("[任务执行失败2]");
}
```

启动项目, 输出结果如下:

```text
[开始执行任务2]
[开始执行任务2]
[开始执行任务2]
[任务执行失败1]
```

```text
[开始执行任务1]
[开始执行任务1]
[开始执行任务1]
[任务执行失败1]
```

这里只是记录了自己测试时的一个简单示例, 项目中的重试机制的地方我也已经使用 spring-retry 进行了重构

# 参考资料

网上已经有很多大佬写了更加深入和详细的用法, 学习的时候参考一些资料, 介绍得很详细, 如下:

* [spring-retry github](https://github.com/spring-projects/spring-retry)
* [Spring Retry 简介](http://iyiguo.net/blog/2016/01/17/spring-retry-simple-introduce/)
* [Spring Retry 常用示例](http://iyiguo.net/blog/2016/01/19/spring-retry-common-case/)
* [Spring 之重试框架 spring-retry](https://vther.github.io/spring-retry/)