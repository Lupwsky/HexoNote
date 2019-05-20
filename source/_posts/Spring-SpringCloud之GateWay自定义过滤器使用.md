---
title: Spring-SpringCloud之GateWay自定义过滤器使用
date: 2019-05-20 22:55:39
categories: Spring
---

GateWay 虽然自带了多种类型的过滤器, 但是在实际的开发中很多功能需求中这些过滤器仍然不能满足需求, 例如实现自定的限流功能, 对请求日志相关信息进行输出等, 这个时候需要自定义过滤器实现

# 自定义网关过滤器

自定义网关过滤器需要实现 GatewayFilter 和 Ordered 两个类中的方法, 分别实现  filter() 方法和 getOrder() 方法, filter() 方法用于实现自定义逻辑, getOrder() 方法表示过滤器默认的优先级

```java
@Slf4j
public class MyGateWayFilter implements GatewayFilter, Ordered {
    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        // pre
        InetSocketAddress remoteAddress = exchange.getRequest().getRemoteAddress();
        String hostName = Optional.ofNullable(remoteAddress)
                .orElse(new InetSocketAddress("000.000.000.000", 0))
                .getHostName();
        log.info("remote address = {}", hostName);

        // post
        return chain.filter(exchange).then(Mono.fromRunnable(() -> {
            exchange.getFormData().hasElement().subscribe(value -> log.info("has element = {}", value));
        }));
    }

    @Override
    public int getOrder() {
        return Ordered.LOWEST_PRECEDENCE;
    }
}
```

<!-- more -->

使用示例如下:

```java
@Bean
public RouteLocator routeLocator(RouteLocatorBuilder builder, RedisRateLimiter redisRateLimiter) {
    return builder.routes()
            .route("cloud-provider0", r -> r.path("/get/username/from/9020")
                    .filters(f -> f.filter(new MyGateWayFilter()))
                    .uri("http://localhost:9020"))
            .build();
}
```

# 自定义全局过滤器

GateWay 除了内置的全局过滤器z之外, 还可以自定义全局过滤器, 全局过滤器会被应用到所有的路由上, 而网关过滤器只会应用到单个路由上, 全局过滤器需要实现 GlobalFilter 和 Ordered 两个类中的方法, 示例如下:

```java
@Slf4j
public class MyGlobalFilter implements GlobalFilter, Ordered {
    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        log.info("my global filter = {}", exchange.getRequest().getRemoteAddress().getHostName());
        return chain.filter(exchange);
    }

    @Override
    public int getOrder() {
        return Ordered.LOWEST_PRECEDENCE;
    }
}
```


# 参考资料


