---
title: SpringCloud-Eureka服务注册与发现
date: 2018-09-05 23:16:15
categories:  Spring
---

# Eureka 服务注册和发现

首先创建一个项目作为所有之后项目的父项目, 添加以下相关依赖

```xml
<!-- 版本管理 -->
<dependencyManagement>
    <dependencies>

        <!-- spring-cloud-dependencies  -->
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-dependencies</artifactId>
            <version>Finchley.RELEASE</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>

        <!-- spring-cloud-starter-ribbon -->
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-ribbon</artifactId>
            <version>1.4.5.RELEASE</version>
        </dependency>
    </dependencies>
</dependencyManagement>
```

<!-- more -->

## 服务注册中心

### 依赖

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-eureka-server</artifactId>
</dependency>
```

### 配置

```ini
spring.application.name=cloud-server0-dev
server.port=9010
# 禁止自注册, 项目仅作为服务注册中心
eureka.client.register-with-eureka=false
# 禁止获取注册信息
eureka.client.fetch-registry=false
```

如果允许服务注册中心自己也是服务提供者或者服务消费者, 一定要指明服务注册中心的地址 (即只要 register-with-eureka 为 true, 就必须指定服务注册中心的地址, 这个地址可以为自身地址), 多个地址以逗号分隔, 按如下所示配置, 另外特别要注意的时注册中心的地址默认的 key 值是 defaultZone

```ini
spring.application.name=cloud-server0-dev
server.port=9010
# 禁止自注册, 项目仅作为服务注册中心
eureka.client.register-with-eureka=true
# 禁止获取注册信息
eureka.client.fetch-registry=true
# 注册中心地址
eureka.client.service-url.defaultZone=http://localhost:9010/eureka/
```

### 启动服务注册中心

在 Application 类加入了 @EnableEurekaServer 注解, 声明这是一个 Eureka 服务器, 其他的没有变化, 如果这样直接启动会出现 ConnectException 异常, 这是由于 Earuka 自注册造成的, 在配置文件中配置相关项可以解决

```java
@SpringBootApplication
@EnableEurekaServer
public class Application {
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}
```

启动成功后即可访问 `http:localhost:9010` Eureka 服务器控制台

## 服务提供者

### 依赖

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-eureka-server</artifactId>
</dependency>
```

### 配置

```ini
server.port=9020
# Eureka 的服务与服务之间相互调用一般都是根据这个应用名
spring.application.name=cloud-provider0-dev
# 配置服务需要注册到的服务注册中心的地址
eureka.client.service-url.defaultZone=http://localhost:9010/eureka/
```

### 实现和启动服务

在 Application 类加入了 @EnableEurekaServer 注解, 声明这是一个 Eureka 客户端, 其他的没有变化

```java
@SpringBootApplication
@EnableEurekaClient
public class Application {
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}
```

```java
@RestController
public class CommonController {
    @GetMapping("/get/username")
    public String getUsername() {
        return "username-from-provider-0";
    }
}
```

启动成功可以在 Eureka 服务器控制台看到注册成功的服务

## 服务调用者

### 依赖

这里除了添加了 eureka client 之外还使用了 eureka ribbon, 服务端需要通过服务名来调用服务和负载均衡需要使用 ribbon

```xml
 <dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
</dependency>

<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-ribbon</artifactId>
</dependency>
```

### 配置

```ini
server.port=9030
spring.application.name=cloud-consumer0-dev
eureka.client.service-url.defaultZone=http://localhost:9010/eureka/
```

### 实现和调用服务

启动类

```java
@SpringBootApplication
@EnableDiscoveryClient
public class Application {
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}
```

接着主要的是 RestTemplate 被 @LoadBalanced 注解修饰后，这个 RestTemplate 实例就就可以解析并根据服务名称来调用对应服务提供的方法

```java
@Configuration
public class RestTemplateConfig {

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
}
```

调用服务提供的方法, 根据服务名称来调用, `http://服务名/get/username` 在 RestTemplate 实例加了 @LoadBalanced 注解后能够被解析和顺利访问对应服务提供的 REST 接口

```java
@RestController
@Slf4j
public class CommonController {
    private final RestTemplate restTemplate;

    @Autowired
    public CommonController(RestTemplate restTemplate) {
        this.restTemplate = restTemplate;
    }

    @PostMapping("/get/username/test")
    public void getUsername() {
        for (int i = 0; i< 10; i++) {
            getAndPrintUsername();
        }
    }

    private void getAndPrintUsername() {
        // 根据应用名称来调用
        String url = "http://cloud-provider0-dev/get/username";
        String username = restTemplate.getForObject(url, String.class);
        log.error("username = {}", username);
    }
}
```

启动项目后, 调用 `/get/username/test` 接口, 可以看到服务被调用, 控制台输出如下:

```text
2018-07-27 15:16:43 [http-nio-9030-exec-1] ERROR com.cloud.controller.CommonController - username = username-from-provider-0
2018-07-27 15:16:43 [http-nio-9030-exec-1] ERROR com.cloud.controller.CommonController - username = username-from-provider-0
2018-07-27 15:16:43 [http-nio-9030-exec-1] ERROR com.cloud.controller.CommonController - username = username-from-provider-0
2018-07-27 15:16:43 [http-nio-9030-exec-1] ERROR com.cloud.controller.CommonController - username = username-from-provider-0
2018-07-27 15:16:43 [http-nio-9030-exec-1] ERROR com.cloud.controller.CommonController - username = username-from-provider-0
2018-07-27 15:16:43 [http-nio-9030-exec-1] ERROR com.cloud.controller.CommonController - username = username-from-provider-0
2018-07-27 15:16:43 [http-nio-9030-exec-1] ERROR com.cloud.controller.CommonController - username = username-from-provider-0
2018-07-27 15:16:43 [http-nio-9030-exec-1] ERROR com.cloud.controller.CommonController - username = username-from-provider-0
2018-07-27 15:16:43 [http-nio-9030-exec-1] ERROR com.cloud.controller.CommonController - username = username-from-provider-0
2018-07-27 15:16:43 [http-nio-9030-exec-1] ERROR com.cloud.controller.CommonController - username = username-from-provider-0
```

# Eureka 服务端集群

这里的例子主要是服务提供者的集群, 我们再添加一个服务提供者的项目, 依赖和配置文件还有应用名 (将作为服务名) 和上面的李子中服务提供者项目几乎一样, 不同的地方是配置文件的端口号更改一下, 变为 9021, `/get/username` 接口返回的值变更为 `username-from-provider-1`, 如下所示:

```ini
server.port=9021
# Eureka 的服务与服务之间相互调用一般都是根据这个应用名
spring.application.name=cloud-provider0-dev
# 配置服务需要注册到的服务注册中心的地址
eureka.client.service-url.defaultZone=http://localhost:9010/eureka/
```

```java
@RestController
public class CommonController {
    @GetMapping("/get/username")
    public String getUsername() {
        return "username-from-provider-1";
    }
}
```

启动服务后, 在 Eureka 控制台就可以看到 `CLOUD-PROVIDER0-DEV` 的服务下 Status 值有两个服务激活了 `UP (2) - v-pwlu-pc0.wesure.cn:cloud-provider0-dev:9020 , v-pwlu-pc0.wesure.cn:cloud-provider0-dev:9021`, 调用服务调用者的 `/get/username/test` 接口, 可以看到服务被调用, 查看输出日志, 可以看到负载均衡也是起到作用了, 控制台输出如下:

```text
2018-07-27 16:50:08 [http-nio-9030-exec-1] ERROR com.cloud.controller.CommonController - username = username-from-provider-0
2018-07-27 16:50:08 [http-nio-9030-exec-1] ERROR com.cloud.controller.CommonController - username = username-from-provider-1
2018-07-27 16:50:08 [http-nio-9030-exec-1] ERROR com.cloud.controller.CommonController - username = username-from-provider-0
2018-07-27 16:50:08 [http-nio-9030-exec-1] ERROR com.cloud.controller.CommonController - username = username-from-provider-1
2018-07-27 16:50:08 [http-nio-9030-exec-1] ERROR com.cloud.controller.CommonController - username = username-from-provider-0
2018-07-27 16:50:08 [http-nio-9030-exec-1] ERROR com.cloud.controller.CommonController - username = username-from-provider-1
2018-07-27 16:50:08 [http-nio-9030-exec-1] ERROR com.cloud.controller.CommonController - username = username-from-provider-0
2018-07-27 16:50:08 [http-nio-9030-exec-1] ERROR com.cloud.controller.CommonController - username = username-from-provider-1
2018-07-27 16:50:08 [http-nio-9030-exec-1] ERROR com.cloud.controller.CommonController - username = username-from-provider-0
2018-07-27 16:50:08 [http-nio-9030-exec-1] ERROR com.cloud.controller.CommonController - username = username-from-provider-1
```

# Eureka 服务注册中心集群

显示的项目中一般除了服务提供端要集群之外, 服务注册中心往往也需要集群, 以避免单机故障问题, 服务注册中心集群的配置文件如下, 一般没有必要创建多个项目, 修改下配置文件, 让地址不同即可, 同时让这些服务注册中心相互注册即可, 配置文件如下, 这里配置三个服务配置中心

```ini
server.port=9010
eureka.instance.hostname=192.168.10.80
eureka.client.serviceUrl.defaultZone=http://192.168.10.81:9011/eureka/,http://192.168.10.82:9012/eureka/

server.port=9011
eureka.instance.hostname=192.168.10.81
eureka.client.serviceUrl.defaultZone=http://192.168.10.80:9010/eureka/,http://192.168.10.82:9012/eureka/

server.port=9012
eureka.instance.hostname=192.168.10.82
eureka.client.serviceUrl.defaultZone=http://192.168.10.80:9090/eureka/,http://192.168.10.81:9011/eureka/
```

除了这里要更改之外, 服务提供者的配置中心地址也需要更改下, 将三个地址都加入, 其实只些一个也可以, 但是这样在某台服务注册中心挂掉用, 就会出现问题, 既然服务注册中心使用了集群, 就添加全部地址吧

```ini
server.port=9030
spring.application.name=cloud-consumer0-dev
eureka.client.service-url.defaultZone=http://192.168.10.80:9010/eureka/,http://192.168.10.81:9011/eureka/,http://192.168.10.82:9012/eureka/
```