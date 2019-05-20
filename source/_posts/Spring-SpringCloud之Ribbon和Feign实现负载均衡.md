---
title: Spring-SpringCloud之Ribbon和Feign实现负载均衡
date: 2019-05-20 22:56:47
categories: Spring
---

在 [SpringCloud-Eureka服务注册与发现](http://antsnote.club/2018/09/05/SpringCloud-Eureka%E6%9C%8D%E5%8A%A1%E6%B3%A8%E5%86%8C%E4%B8%8E%E5%8F%91%E7%8E%B0/) 一节中, 使用 Euraka 实现了服务注册于发现后, 下一步就需要服务消费者去调用服务提供者提供的接口了, Spring Cloud 的微服务之间是使用 HTTP 调用的方式实现的, 一个服务调用另一个服务的 HTTP 接口示例如下:

```java
private final RestTemplate restTemplate;

// 服务消费者 (IP = 192.168.10.156) 调用服务提供者 (IP = 192.168.10.157) 的 URL
String url = "http://192.168.10.157:9020/get/username";
String username = restTemplate.getForObject(url, String.class);
log.info("username = {}", username);
```

<!-- more -->

# LoadBalancerClient

上面的一个服务调用另一个服务的 HTTP 接口是又两个问题需要解决的, 第一个是直接指定 URL 中强制写死了 IP 和端口号, 在 IP 或端口号发生变化后, 所有的服务消费端都需要改动代码中写死的 IP, 第二个是如果服务提供端启动了多个实例, 上面调用的方式并没有实现负载均衡, 因为 IP 已经写死了, 上面的写法在生产环境下肯定是不适用的, 需要进行改进, 在 Spring Cloud Commons 包下面定义有很多服务治理相关的接口和功能实现类, LoadBalancerClient 就是其中的一个, 利用 LoadBalancerClient 类可以方便的解决上面的问题, 不需要强制写死 IP, 只需要指定调用哪一个服务名即可实现客户端版的负载均衡, 使用示例如下:

```java
@Autowired
private LoadBalancerClient loadBalancerClient;

@Autowired
private RestTemplate restTemplate;

// 指定需要调用的服务名即可完成客户端的负载均衡
ServiceInstance serviceInstance = loadBalancerClient.choose("cloud-provider0-dev");
String url = "http://" + serviceInstance.getServiceId() + ":" + serviceInstance.getPort() + "/get/username";
log.info("url = {}", url);
String username = restTemplate.getForObject(url, String.class);
log.info("username = {}", username);
```

# Ribbon

Ribbon 是对 LoadBalancerClient 进行一层封装, 在直接使用 LoadBalancerClient 的时候, 我们需要注入 LoadBalancerClient 实例, 然后通过他的 choose 选择对应的服务, 里面的实现是获取到 Eureka 服务注册中心中注册的服务名和服务端口号, 最后通过负载均衡算法获取服务中的其中一个 IP, 把 URL 中的服务名替换成 IP 后再使用 RestTemplate 实例进行调用, Ribbon 可以让我们更方便的实现问服务之间的调用和客户端负载均衡的实现, 不需要在注入 LoadBalancerClient 实例, 仅需要在定义 RestTemplate Bean 的时候添加 @LoadBalanced 注解, 使用实例如下:

首先添加依赖:

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-ribbon</artifactId>
</dependency>
```

定义 RestTemplate 类型的 Bean 的时候添加 @LoadBalanced 注解:

```java
@Bean
@LoadBalanced
public RestTemplate restTemplate(ClientHttpRequestFactory factory) {
    return new RestTemplate(factory);
}

@Bean
public ClientHttpRequestFactory simpleClientHttpRequestFactory() {
    SimpleClientHttpRequestFactory factory = new SimpleClientHttpRequestFactory();
    factory.setReadTimeout(5000);
    factory.setConnectTimeout(15000);
    return factory;
}
```

调用的时候, URL 中直接使用服务名即可, 使用示例如下:

```java
// URL = "http://[服务名]/get/username"
String username = restTemplate.getForObject("http://cloud-provider0-dev/get/username", String.class);
log.info("username = {}", username);
```

和 RPC 相比较, RPC 的体验是调用远程服务的方法和调用本地方法体验是一样的, 虽然 Ribbon 让调用服务的接口已经很方便了, 但还是有些不足, 不能在调用的时候无法拥有和调用本地方法一样的体验, 需要我们在服务消费者这一段注入 RestTempalte 的实例, 然后通过 URL 去调用相关接口, 因此出现对 Ribbon 进行了进一步的封装的开源库 Feign, 使得微服务之间的方法调用接近在本地调用方法一样, Feign 的使用方法见下面 Fegin 这一小节

# Fegin

首先需要引入依赖, 需要注意的是, 由于 Fegin  是对 Ribbon 的封装, Fegin 里面已经依赖了 Ribbon, 因此就没有必要再引入 Ribbon 了, 如下:

```xml
<!-- 引入了 feign 就不需要再引入 ribbon 了-->
<!-- <dependency>-->
<!--     <groupId>org.springframework.cloud</groupId>-->
<!--     <artifactId>spring-cloud-starter-ribbon</artifactId>-->
<!-- </dependency>-->

<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-openfeign</artifactId>
</dependency>
```

使用 Fegin 需要在启动类添加 @EnableFeignClients 注解, 如下:

```java
@SpringBootApplication
@EnableDiscoveryClient
@EnableFeignClients
public class Application {
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}
```

在服务提供者 (服务名 = cloud-provider0-dev) 处提供一个接口, 如下:

```java
@GetMapping("/get/username")
public String getUsername() {
    return "username-from-provider-0";
}
```

在服务消费者处定义一个名为 ApiInterface 的接口, 并添加 @FeignClient("服务名") 注解, 如下:

```java
@FeignClient("cloud-provider0-dev")
public interface ApiInterface {
    
    // 这里的 URL 就是服务调用时所需要的 URL
    @GetMapping("/get/username")
    String getUserName();
}
```

在服务端注入 ApiInterface 实例, 并调用里面的方法, 就和在本地调用方法体验一致, 如下:

```java
@Autowired
private ApiInterface apiInterface;

@GetMapping("/get/username/with/fegin")
public String getUserName() {
    return apiInterface.getUserName();
}
```

Feign 通过 @FeignClient("服务名") 注解和方法上的 URL 就可以知道需要调用的时哪个服务和 URL 信息, Fegin 里面使用 Ribbon 的那一套方法去调用 HTTP 接口, 实际上像 ApiInterface 这种对外提供服务的接口需要封装并上传到公司的私有 Maven 仓库中, 在服务提供端和消费端添加这个依赖的方式可以更加灵活管理对外提供的服务

# 参考资料

* [Ribbon 的注解使用报错 -- No instances available for IP](https://blog.csdn.net/november22/article/details/54612454)
* [Spring Cloud 构建微服务架构 : 服务消费 (Ribbon) 【Dalston版】](http://blog.didispace.com/spring-cloud-starter-dalston-2-2/)
