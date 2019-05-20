---
title: Spring-SpringCloud之GateWay入门和Predicate
date: 2019-05-20 22:51:01
categories: Spring
---

# Spring Cloud Gateway 和 Zuul 的性能比较

* [Spring Cloud Gateway 真的有那么差吗？](https://blog.csdn.net/j3T9Z7H/article/details/79295104)

# 基础使用

## 引入依赖

在端口号为 9030 的项目中引入如下依赖

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-gateway</artifactId>
</dependency>
```

<!-- more -->

## 启动项目, 端口号为 9020

项目里面定义了 URL = /get/username/from/9010

```java
@GetMapping("/get/username/from/9020")
public String getUsernameFrom9020() {
    return "username-from-9020";
}
```

## 启动项目, 端口号 9021

项目里面定义了 URL = /get/username/from/9021

```java
@GetMapping("/get/username/from/9021")
public String getUsernameFrom9021() {
    return "username-from-9021";
}
```

## 在 9030 的项目中设置路由规则

```java
@Configuration
public class RouteLocatorConfig {
    @Bean
    public RouteLocator routeLocator(RouteLocatorBuilder builder) {
        return builder.routes()
                .route(r -> r.path("/get/username/from/9020").uri("http://localhost:9020"))
                .route(r -> r.path("/get/username/from/9021").uri("http://localhost:9021"))
                .build();
    }
}
```

经过上面的配置完成后, 当我们访问 `http://localhost:9030/get/username/from/9020` 时就会转发到 `http://localhost:9020/get/username/from/9020`, 而访问 `http://localhost:9030/get/username/from/9021` 时就会转发到 `http://localhost:9021/get/username/from/9021` 上

## 开始测试

启动端口号为 9030 项目提示出现如下错误:

```text
Spring MVC found on classpath, which is incompatible with Spring Cloud Gateway at this time.
Please remove spring-boot-starter-web dependency.

Parameter 0 of method modifyRequestBodyGatewayFilterFactory 
in org.springframework.cloud.gateway.config.GatewayAutoConfiguration 
required a bean of type 'org.springframework.http.codec.ServerCodecConfigurer' 
that could not be found
```

由于 9030 的项目中之前引入了 spring-boot-starter-web, 产生了冲突, 将 spring-boot-starter-web 依赖去掉即可, 启动后测试正常, 调用 `http://localhost:9030/get/username/from/9020` 接口时, 返回 `username-from-9020`, 调用 `http://localhost:9030/get/username/from/9021` 接口时, 返回 `username-from-9021`

# 断言 (Predicate)

GateWay 中的核心 Route (路由) 由 Predicate (断言) 和 Filter (过滤器) 两部分组成, 一个请求能不能匹配路由规则主要由断言决定, 断言成功则路由匹配, GateWay 默认提供了一些断言规则, 如下

## After Route Predicate

在指定时间之后访问时路由才会匹配请求

```java
@Bean
public RouteLocator routeLocator(RouteLocatorBuilder builder) {
    return builder.routes()
            .route(r -> r.path("/get/username/from/9020")
                    .and()
                    .after(ZonedDateTime.of(LocalDateTime.of(2019, 5, 14, 0, 0, 0), ZoneId.systemDefault()))
                    .uri("http://localhost:9020"))
            .build();
}
```

在 2019-05-14 00:00:00 之后访问才有效, 例如在 2019-05-13 00:00:00 时去访问, 在 PostMan 则会出现如下 404 错误提示:

```json
{
    "timestamp": "2019-05-13T08:38:59.018+0000",
    "path": "/get/username/from/9020",
    "status": 404,
    "error": "Not Found",
    "message": null
}
```

## Before Route Predicate

在指定时间之前访问时路由才会匹配请求

```java
@Bean
public RouteLocator routeLocator(RouteLocatorBuilder builder) {
    return builder.routes()
            .route(r -> r.path("/get/username/from/9020")
                    .and()
                    .before(ZonedDateTime.of(LocalDateTime.of(2019, 5, 14, 0, 0, 0), ZoneId.systemDefault()))
                    .uri("http://localhost:9020"))
            .build();
}
```

## Between Route Predicate

在两个时间之间访问时路由才会匹配请求

```java
@Bean
public RouteLocator routeLocator(RouteLocatorBuilder builder) {
    return builder.routes()
            .route(r -> r.path("/get/username/from/9020")
                    .and()
                    .between(ZonedDateTime.of(LocalDateTime.of(2019, 5, 14, 0, 0, 0), ZoneId.systemDefault()),
                             ZonedDateTime.of(LocalDateTime.of(2019, 5, 20, 0, 0, 0), ZoneId.systemDefault()))
                    .uri("http://localhost:9020"))
            .build();
}
```

## Cookie Route Predicate

请求中需要携带指定的 cookie, 并且 cookie 的值匹配正则表达式才能匹配到路由

```java
@Bean
public RouteLocator routeLocator(RouteLocatorBuilder builder) {
    // 这个例子要求 cookie 的名为 username, 并且值为 lpw 才能匹配
    return builder.routes()
            .route(r -> r.path("/get/username/from/9020")
                    .and()
                    .cookie("username", "lpw")
                    .uri("http://localhost:9020"))
            .build();
}
```

## Header Route Predicate

和 Cookie Route Predicate 类似, 需要请求头携带指定的头部信息, 如果带有值, 则值匹配正则表达式才能匹配到路由

```java
@Bean
public RouteLocator routeLocator(RouteLocatorBuilder builder) {
    return builder.routes()
            .route(r -> r.path("/get/username/from/9020")
                    .and()
                    .header("username", "lpw")
                    .uri("http://localhost:9020"))
            .build();
}
```

## Host Route Predicate

符合域名匹配规则要求的才能匹配路由, 如下例中只要是 `www.smc-system.` 开头的域名都能匹配成功

```java
 @Bean
public RouteLocator routeLocator(RouteLocatorBuilder builder) {
    return builder.routes()
            .route(r -> r.path("/get/username/from/9020")
                    .and()
                    .host("www.smc-system.**")
                    .uri("http://localhost:9020"))
            .build();
}
```

## Method Route Predicate

通过 HTTP 请求的 method 来匹配, 如下例只匹配 GET 请求或者 POST 请求等等

```java
@Bean
public RouteLocator routeLocator(RouteLocatorBuilder builder) {
    return builder.routes()
            .route(r -> r.path("/get/username/from/9020")
                    .and()
                    .method(HttpMethod.GET)
                    .or()
                    .method(HttpMethod.POST)
                    .uri("http://localhost:9020"))
            .build();
}
```

## Path Route Predicate

通过 path 来匹配, 例如下面的例子, 所有以路径 `/get/username/` 开头的 (包括 /get/username/) 请求都可以匹配

```java
@Bean
public RouteLocator routeLocator(RouteLocatorBuilder builder) {
    return builder.routes()
            .route(r -> r.path("/get/username/from/9020")
                    .and()
                    .path("/get/username/**")
                    .uri("http://localhost:9020"))
            .build();
}
```

path 匹配使用的是 PathMatcher 实现的, 参考资料: [SpringMVC 路径匹配规则 AntPathMatcher](https://blog.csdn.net/qq_21251983/article/details/53034425)

# Query Route Predicate

通过请求携带的参数俩匹配路由

```java
@Bean
public RouteLocator routeLocator(RouteLocatorBuilder builder) {
    return builder.routes()
            .route(r -> r.path("/get/username/from/9020")
                    .and()
                    .query("username", "lpw")
                    .uri("http://localhost:9020"))
            .build();
}
```

# RemoteAddr Route Predicate

通过访问的 IP 来匹配

```java
@Bean
public RouteLocator routeLocator(RouteLocatorBuilder builder) {
    return builder.routes()
            .route(r -> r.path("/**")
                    .and()
                    .remoteAddr("10.98.164.127")
                    .uri("http://localhost:9020"))
            .build();
}
```

本机 IP = 10.98.164.127, 当使用远程地址断言的时候, 只有 `http://10.98.164.127:9030` 的才能匹配, 而 `http://localhost:9030` 或者 `http://127.0.0.1:9030` 都是匹配不上的
