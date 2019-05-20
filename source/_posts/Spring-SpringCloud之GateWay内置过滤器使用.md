---
title: Spring-SpringCloud之GateWay内置过滤器使用
date: 2019-05-20 22:53:04
categories: Spring
---

GateWay 中的核心 Route (路由) 由 Predicate (断言) 和 Filter (过滤器) 两部分组成, 可以通过过滤器实现对 HTTP 请求输入和输出的更改, 如更改 Header 和 Parameter 等, 通常在项目中使用过滤器来完成用户权限验证和非业务性质的校验等工作, GateWay 默认提供了一些默认的过滤器

# AddRequestHeader GatewayFilter 和 AddRequestParameter GatewayFilter

这两个过滤器用于添加和修改 HTTP 请求的 Header 和 Paramter, 使用示例如下:

```java
@Bean
public RouteLocator routeLocator(RouteLocatorBuilder builder) {
    return builder.routes()
            .route(r -> r.path("/get/username/from/9020")
                    .filters(gatewayFilterSpec -> gatewayFilterSpec.stripPrefix(1)
                            .addRequestHeader("MY_HEADER", "lpw")
                            .addRequestParameter("MY_PARAM", "test"))
                    .uri("http://localhost:9020"))
            .build();
}
```

<!-- more -->

可以在 Rest 接口中通过 HttpServletRequest 的实例来获取 Header 和 Paramter, 示例如下:

```java
@GetMapping("/get/username/from/9020")
public String getUsernameFrom9020(HttpServletRequest request) {
    String myHeader = request.getHeader("MY_HEADER");
    log.info("myHeader = {}", myHeader);

    String myParam = request.getParameter("MY_PARAM");
    log.info("myParam = {}", myParam);
    return "username-from-9020";
}
```

访问 `http://localhost:9030/get/username/from/9020` 输出结果如下, 注意如果直接访问 `http://localhost:9020/get/username/from/9020` 是获取不到值的:

```text
[http-nio-9020-exec-1] INFO  com.cloud.controller.CommonController - myHeader = myHeaderValue
[http-nio-9020-exec-1] INFO  com.cloud.controller.CommonController - myParam = myParam
```

# AddResponseHeader GatewayFilter

添加和修改 HTTP 请求的响应 Header

```java
@Bean
public RouteLocator routeLocator(RouteLocatorBuilder builder) {
    return builder.routes()
            .route(r -> r.path("/get/username/from/9020")
                    .filters(gatewayFilterSpec -> gatewayFilterSpec.addResponseHeader("MY_RESP_HEADER", "test"))
                    .uri("http://localhost:9020"))
            .build();
}
```

在 PostMan 中测试, 可以看到设置的 Header, 如下所示:

```text
MY_RES_HEADER → myResHeader
Content-Type → text/plain;charset=UTF-8
Content-Length → 18
Date → Wed, 15 May 2019 09:43:13 GMT
```

# RemoveRequestHeader GatewayFilter 和 RemoveResponseHeader GatewayFilter

既然能否添加请求的 Header 和添加请求响应的 Header, 那么也有删除请求的 Header 和删除请求响应的 Header 的过滤器, RemoveRequestHeader GatewayFilter 和 RemoveResponseHeader GatewayFilter 可以完成此类功能需求, 使用方法和 AddRequestHeader GatewayFilter, AddResponseHeader GatewayFilter 一样, 就不贴出测试代码了

# SetRequestHeader GatewayFilter 和 SetResponseHeader GatewayFilter

用于修改请求的 Header 和添加请求响应的 Header, 使用方法和 AddRequestHeader GatewayFilter, AddResponseHeader GatewayFilter 一样, 就不贴出测试代码了

# PrefixPath GatewayFilter

用于设置路径前缀, 使用示例如下:

```java
@Bean
public RouteLocator routeLocator(RouteLocatorBuilder builder) {
return builder.routes()
        .route("cloud-provider0", r -> r.path("/username/from/9020")
                .filters(f -> f.prefixPath("/get"))
                .uri("http://localhost:9020"))
        .build();
}
```

当我们访问 `http://localhost:9030/username/from/9020`, 会转换成访问 `http://localhost:9020/get/username/from/9020`

# StripPrefix GatewayFilter

用于设置路径前缀的过滤器, 也有删除前缀的过滤器, 使用示例如下:

```java
@Bean
public RouteLocator routeLocator(RouteLocatorBuilder builder) {
return builder.routes()
        .route("cloud-provider0", r -> r.path("/test/test/get/username/from/9020")
                .filters(f -> f.stripPrefix(2))
                .uri("http://localhost:9020"))
        .build();
}
```

如上配置, 使用 `http://localhost:9030/test/test/get/username/from/9020`, 会去除 `/test/test` 两个路径前缀, 会转换成访问 `http://localhost:9020/get/username/from/9020`

# RedirectTo GatewayFilter

在出现指定错误错误状态码时会重定向访问指定的 URL, 其中 status 必须是 300 系列的 HTTP 状态码, 否则启动报错, 错误中也会提示你必须是 300 系列的状态码

```java
@Bean
public RouteLocator routeLocator(RouteLocatorBuilder builder) {
return builder.routes()
        .route("cloud-provider0", r -> r.path("/get/username/from/9020")
                .filters(f -> f.redirect(302, "http://localhost:9020/get/username"))
                .uri("http://localhost:9030/not/exist"))
        .build();
}
```

# RewritePath GatewayFilter

RewritePath 过滤器可以使用正则表达式对路径进行重写, 使用示例如下:

```java
@Bean
public RouteLocator routeLocator(RouteLocatorBuilder builder) {
return builder.routes()
        .route("cloud-provider0", r -> r.path("/get/username/test")
                .filters(f -> f.rewritePath("/get/username/test", "/get/username/from/9020"))
                .uri("http://localhost:9020"))
        .build();
}
```

这个例子中, 使用 `http://localhost:9030/get/username/test` 访问时, `/get/username/test` 会被替换成 `/get/username/from/9020`, 最后被路由到 `http://localhost:9020/get/username/from/9020`

# SetPath GatewayFilter

相比 RewritePath 过滤器, setPath 过滤器就比较粗暴了, 直接就替换整个路径, 使用示例如下:

```java
@Bean
public RouteLocator routeLocator(RouteLocatorBuilder builder) {
return builder.routes()
        .route("cloud-provider0", r -> r.path("/get/username/from/9020")
                .filters(f -> f.setPath("/test"))
                .uri("http://localhost:9020"))
        .build();
}
```

在 PostMan 中访问接口 `http://localhost:9030/get/username/from/9020`, 会提示找不到 `/test` 的路径, 在 PostMan 中输出的信息如下:

```text
{
    "timestamp": "2019-05-15T11:56:32.150+0000",
    "status": 404,
    "error": "Not Found",
    "message": "No message available",
    "path": "/test"
}
```

# SetStatus GatewayFilter

SetStatus 过滤器用于更改返回的响应码, 使用示例如下:

```java
@Bean
public RouteLocator routeLocator(RouteLocatorBuilder builder) {
return builder.routes()
        .route("cloud-provider0", r -> r.path("/get/username/from/9020")
                .filters(f -> f.setStatus(400))
                .uri("http://localhost:9020"))
        .build();
}
```

在 PostMan 中访问 `http://localhost:9030/get/username/from/9020` 时, 访问成功获取到数据, 但是可以看到返回的状态码是更改后的 400, 如下:

```text
400 Bad Request 
Time:166 ms 
Size:143 B
```

# Retry GatewayFilter

Retry 过滤器可以设置访问接口出错时进行重试, 可以设置进行重试的次数, 状态码和请求方法等条件, 使用示例如下:

```java
@Bean
public RouteLocator routeLocator(RouteLocatorBuilder builder) {
return builder.routes()
        .route("cloud-provider0", r -> r.path("/get/username/from/9020")
                .filters(f -> f.retry(retryConfig -> {
                        retryConfig.setRetries(3) // 设置重试次数
                                .setStatuses(HttpStatus.BAD_REQUEST, HttpStatus.NOT_FOUND) // 需要重试的状态码
                                .setMethods(HttpMethod.GET) // 设置需要重试的方法
                                .setExceptions(FileNotFoundException.class); // 设置需要重试的异常 
                }))
                .uri("http://localhost:9020"))
        .build();
}
```

# Modify Request Body GatewayFilter 和 Modify Response Body GatewayFilter

这两个过滤器可以修改请求体和响应体的内容, 但都是 beta 版本, 查询网上的一些测试资料, 验证的结果表面有很大的性能问题, 因此在项目中也没有轻易的使用

```java
@Bean
public RouteLocator routeLocator(RouteLocatorBuilder builder) {
return builder.routes()
        .route("cloud-provider0", r -> r.path("/get/username/from/9020")
                .filters(f -> f.modifyRequestBody(String.class, UserInfoParams.class, new RewriteFunction<String, UserInfoParams>() {
                        @Override
                        public Publisher<UserInfoParams> apply(ServerWebExchange serverWebExchange, String s) {
                        return Mono.just(UserInfoParams.builder().username(s).build());
                        }
                }))
                .uri("http://localhost:9020"))
        .build();
}

@Bean
public RouteLocator routeLocator(RouteLocatorBuilder builder) {
return builder.routes()
        .route("cloud-provider0", r -> r.path("/get/username/from/9020")
                .filters(f -> f.modifyResponseBody(String.class,
                        UserInfoParams.class,
                        (serverWebExchange, s) -> {
                                return Mono.just(UserInfoParams.builder().username(s).build());
                        }).addResponseHeader(HttpHeaderNames.CONTENT_TYPE.toString(), MediaType.TEXT_PLAIN_VALUE))
                .uri("http://localhost:9020"))
        .build();
}
```

# RequestRateLimiter GatewayFilter

对请求进行限速, 默认使用令牌桶实现, 也可以自定义 Filter 实现限速, 内置的限流过滤器, 依赖 Redis, 需要添加 Redis 依赖, 如下:

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-redis</artifactId>
</dependency>
```


对 Redis 进行配置:

```ini
spring.redis.host=127.0.0.1
spring.redis.port=6379
spring.redis.database=0
spring.redis.password=lupengwei.4585
```

有的文章 Redis 依赖如下, 我测试的时候启动报错了:

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-boot-starter-data-redis-reactive</artifactId>
</dependency>
```

报错内容如下:

```text
***************************
APPLICATION FAILED TO START
***************************

Description:
Parameter 0 of method cachedCompositeRouteLocator in org.springframework.cloud.gateway.config.GatewayAutoConfiguration required a bean of type 'org.springframework.cloud.gateway.filter.factory.RequestRateLimiterGatewayFilterFactory' that could not be found.
- Bean method 'requestRateLimiterGatewayFilterFactory' in 'GatewayAutoConfiguration' not loaded because @ConditionalOnBean (types: org.springframework.cloud.gateway.filter.ratelimit.RateLimiter,org.springframework.cloud.gateway.filter.ratelimit.KeyResolver; SearchStrategy: all) did not find any beans of type org.springframework.cloud.gateway.filter.ratelimit.RateLimiter, org.springframework.cloud.gateway.filter.ratelimit.KeyResolver

Action:
Consider revisiting the conditions above or defining a bean of type 'org.springframework.cloud.gateway.filter.factory.RequestRateLimiterGatewayFilterFactory' in your configuration.
```

在按照错误的提示配置了各种的 Bean 之外, 项目可以正常启动, 但是访问的时还是报 `RedisRateLimiter is not initialized` 错误, 将依赖更换后就正常了, 最终的测试代码如下:

```java
@Bean
RedisRateLimiter redisRateLimiter() {
    // defaultReplenishRate = 每秒处理请求数, defaultBurstCapacity = 令牌桶数量
    return new RedisRateLimiter(1, 100);
}

@Bean
public RouteLocator routeLocator(RouteLocatorBuilder builder, RedisRateLimiter redisRateLimiter) {
return builder.routes()
        .route("cloud-provider0", r -> r.path("/get/username/from/9020")
                .filters(f -> f.requestRateLimiter(config -> config
                        .setRateLimiter(redisRateLimiter)
                        .setKeyResolver((KeyResolver) exchange -> {
                                // 使用 IP 进行限流
                                return Mono.just(exchange.getRequest().getRemoteAddress().getHostName());
                        })))
                .uri("http://localhost:9020"))
        .build();
}
```

在 redis 中也可以看见 `request_rate_limiter.{0:0:0:0:0:0:0:1}` 之类的数据, 具体 redis 是如何保存数据, 可以在 GateWay 的 core 包的 `META-INF/scripts/request_rate_limiter.lua` 文件中去了解

# 参考资料

* [Spring Cloud GateWay-过滤器 GatewayFilter、GlobalFilter、GatewayFilterChain、作用、生命周期、GatewayFilterFactory 内置过滤器](https://www.cnblogs.com/bjlhx/p/9786478.html)
* [Spring Cloud GateWay 学习 (四) Filter](https://blog.csdn.net/fayeyiwang/article/details/86539644)
* [Spring Cloud GateWay 读取 Request Body 数据](https://my.oschina.net/zcqshine/blog/2875060)
* [Spring Cloud (十九) : Spring Cloud Gateway (读取、修改 Request Body)](https://windmt.com/2019/01/16/spring-cloud-19-spring-cloud-gateway-read-and-modify-request-body/)
* [Spring Cloud GateWay 的性能测试](https://www.cnblogs.com/garfieldcgf/p/10484119.html)
* [Spring Cloud(十五) : Spring Cloud Gateway (限流)](https://windmt.com/2018/05/09/spring-cloud-15-spring-cloud-gateway-ratelimiter/)