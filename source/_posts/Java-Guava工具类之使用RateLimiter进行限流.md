---
title: Java-Guava工具类之使用RateLimiter进行限流
date: 2019-05-21 14:59:54
categories: Java
---

在大数据量高并发访问时, 经常会出现服务或接口面对大量请求而不可用的情况, 甚至引发连锁反映导致整个系统崩溃, 限流的技术手段可以很好的应对这种情况, 当请求达到一定的并发数或速率, 就进行等待, 排队, 降级, 拒绝服务等策略, 虽然会影响到用户的体验, 但是可以有效的保证系统不崩溃, 之前有使用 Redis 实现过一个简单的限流, 这里学习下 Guava 的 RateLimiter 的使用方法 

# 使用方法

常见的限流算法有, 令牌桶, 漏斗和计数器算法, 这里就不去详细的介绍了, 下面的参考资料里面也有, Guava 的 RateLimiter 使用的是令牌桶方式实现的限流, 令牌桶限流有一个参数令牌生成速率是我们需要注意的, 其令牌桶的大小和生成速率也是相关的, 如设置速率为每秒生成 5 个令牌, 那么桶的最大值也为 5, 桶中可以保存的最大令牌数永远不会超过桶的大小, 创建一个 RateLimiter 示例如下所示:

```java
@Configuration
public class GuavaRateLimiterConfig {
    @Bean
    public RateLimiter rateLimiter() {
        // 每秒生成 5 个令牌
        return  RateLimiter.create(5);
    }
}
```

<!-- more -->

上面的方式可以生一个限流器, 以恒定的速度每秒生成 5 个令牌, 在使用的时候, 先去获取令牌, 根据是否能获取到令牌在决定是否将服务降级处理, RateLimiter 提供了一些方法用于帮助判断, 如下:

| 方法 | 作用 |
| --- | --- |
| acquire() | 获取一个令牌, 改方法会阻塞直到获取到这一个令牌, 返回值为获取到这个令牌花费的时间 |
| acquire(int permits) | 获取指定数量的令牌, 该方法也会阻塞, 返回值为获取到这 N 个令牌花费的时间 |
| tryAcquire() | 判断时候能获取到令牌, 如果不能获取立即返回 false |
| tryAcquire(int permits) | 获取指定数量的令牌, 如果不能获取立即返回 false |
| tryAcquire(long timeout, TimeUnit unit) | 判断能否在指定时间内获取到令牌, 如果不能获取立即返回 false |
| tryAcquire(int permits, long timeout, TimeUnit unit) | 同上 |

测试代码如下所示:

```java
if (rateLimiter.tryAcquire(1)) {
    log.info("尝试获取令牌成功");
    log.info("获取令牌 = {}", rateLimiter.acquire());
} else {
    log.info("尝试获取令牌失败");
}
```

上面的 RateLimiter 方法生成的 RateLimiter 实例是 SmoothBursty 类型的限流器, Guava有两种限流器, 一种为稳定模式 (SmoothBursty), 令牌生成速度恒定), 一种为渐进模式(SmoothWarmingUp), 令牌生成速度缓慢提升直到维持在一个稳定值, 如果需要获取渐进模式的限流器, 可以调用 RateLimiter 的另一个 create 方法, 如下所示:

```java
@Configuration
public class GuavaRateLimiterConfig {
    @Bean
    public RateLimiter rateLimiter() {
        // 每秒多生成 1 个令牌, 直到稳定后每秒生成 5 个令牌
        // 即第 1 秒生成 1 个令牌, 第 2 秒生成 2 个令牌, 第 3 秒生成 3 个令牌, 直到第 5 秒生成 5 个令牌到达稳定状态
        return  RateLimiter.create(5, 1, TimeUnit.SECONDS);
    }
}
```

上面的例子介绍了使用 Guava 的 RateLimiter 实现简单的限流的思路, 但是现在的后台服务很多基本都是使用的都是集群部署的, Guava 的 RateLimiter 在单服务下使用没有问题, 但是在集群模式下就会出现问题了, 对于集群部署的服务如果需要使用限流还是得用 redis + lua 来实现

# 参考资料

* [使用 Guava RateLimiter 限流以及源码解析](https://segmentfault.com/a/1190000016182737) - 常用限流算法和 Guava 基础使用
* [Guava RateLimiter 源码解析](https://segmentfault.com/a/1190000012875897)
* [Guava 官方文档 RateLimiter 类](http://ifeve.com/guava-ratelimiter/)