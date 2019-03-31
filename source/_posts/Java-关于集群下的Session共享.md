---
title: Java-关于集群下的Session共享
date: 2019-02-20 23:36:37
categories: Java
---

Seesion 实现共享的方案有很多, 主要分为客户端实现和服务器端实现两种, 客户端实现主要是利用 Cookie 保存 Session 数据到客户端, 服务器端的实现有诸如利用 Tomcat 提供的 Session 共享功能, 将 Session 保存到数据库, 或者使用 Redis 等来实现 Session 的共享, 这里记录一些了解到的常见方案

# 客户端 Cookie 保存方式实现 Session 共享

客户端 Cookie 保存实现, 即在服务器端将保存在保存在服务器中 Session 中的数据写入到 Cookie 后保存到客户端 (通常为浏览器) 中, 由于 Session 数据保存在客户端中吗, 所以每次请求都会带上这些 Session 数据, 因此即使两次请求在不同的服务器上也可以获取到 Session 数据, 以此达到 Session 共享, 使用这种方法有以下优缺点:

* 优点一: 可以大大减轻的服务器的压力, 服务器架构也相应变得简单
* 缺点一: Cookie 数据长度和 Cookie 保存的数量有限制
* 缺点二: 每次请求需要带额外的 Cookie 信息, 增加网络带宽
* 缺点三: 客户端 (如浏览器) 可能会禁用 Cookie 功能或者认为的更改 Cookie 的值, 由此带来功能上的问题
* 缺点四: 信息保存在客户端会带来安全问题, 信息的加密和解密, 会带来额外开发和性能开销

<!-- more -->

下面是 Java 设置 Cookie 和获取 Cookie 的例子:

```java
// 设置 Cookie
@GetMapping(value = "/session/cookie/set/test")
public HashMap<String, Object> sessionSetTest(HttpServletRequest request, HttpServletResponse response) {
    // Session 中的数据保存到 Cookie
    Cookie cookie = new Cookie("data", "lupengwei:");

    // 表示哪些请求路径可以获取到这个 Cookie,
    // 例如 JSESSIONID 这个 cookie 的 path 为 "/", 表示所有的同一个 domain 下的所有请求都可以获取到这个 cookie
    // 使用 cookie.setDomain() 方法可以设置 domain
    // 如果没有设置, 默认设置为当前请求的 BASE_URL, 如这个例子 path=/session/cookie/set;
    // 只有 domain + path + name 全匹配才能获取到这个 cookie
    // 因此在开发时需要考虑二级域名等问题
    cookie.setPath("/");
    // 过期时间
    cookie.setMaxAge(3600 * 1000 * 24);
    // 表示仅 HTTP 请求可以获取到这个 cookie
    cookie.setHttpOnly(true);
    response.addCookie(cookie);

    // 在浏览的 cookie 中看到类似如下数据
    // data=lupengwei;
    // path=/;
    // domain=localhost;
    // Expires=Wed, 20 Feb 2019 07:18:29 GMT;
    HashMap<String, Object> resultMap = Maps.newHashMap();
    resultMap.put("sessionId", request.getSession(true).getId());
    return resultMap;
}

// 获取 Cookie
@GetMapping(value = "/session/cookie/get/test")
public JSONObject sessionGetTest(HttpServletRequest request) {
    String cookieValue = "";
    Cookie[] cookies = request.getCookies();
    for (Cookie cookie : cookies) {
        if(Objects.equals(cookie.getName(), "data")) {
            cookieValue = cookie.getValue();
        }
    }
    JSONObject resultData = new JSONObject();
    resultData.put("data", cookieValue);
    return resultData;
}
```

参考资料:

* [Cookie 与 Session](https://uule.iteye.com/blog/2211575)
* [Cookie 详解](https://www.cnblogs.com/z941030/p/4742188.html)

# Tomcat 集群自带的 Session 复制功能实现 Session 共享

Tomcat 集群之间可以通过组播的方式实现集群内的 Session 共享, 由于是 WEB 容器自带的, 所有配置起来比较简单, 修改 Tomcat 容器的配置文件即可, 但是由于通过组播方式同步, 当其中一台机器的的 Session 数据发生变化时, 会分发到其他的机器来进行同步, 会造成一定的网络和性能开销, 集群越大, 开销越大, 不能线性的扩展, 因此这种方案只适合小型网站, 由于自己的实际项目都是使用 SpringBoot, 线上项目以 jar 包的形式启动, 也很少去更改内嵌的 Tomcat 的配置文件, 且 SpringBoot 有其他更好的的方法来实现集群共享, 就没有去实际的操作, 参考资料: [Nginx + Tomcat 实现 Session 共享](http://blog.51cto.com/yw666/1888747)

# 使用缓存实现 Session 共享

目前比较主流的 Session 方式还是基于分布式缓存 Memcached 或 Redis 实现, 具体实现方式主要有 `memcached-session-manager`, `tomcat-redis-session-manager` 和 `Spring Session`, 前两者需要依赖 WEB 容器, 而后者不需要, 在项目中使用的也是 `Spring Session` + `Redis` 来实现的, 这里简单记录下实现方式

添加依赖:

```xml
<dependency>
    <groupId>io.lettuce</groupId>
    <artifactId>lettuce-core</artifactId>
    <version>5.1.3.RELEASE</version>
</dependency>

<dependency>
    <groupId>org.springframework.session</groupId>
    <artifactId>spring-session-data-redis</artifactId>
</dependency>
```

添加配置项:

```ini
# 使用 Redis 存放 Session 信息, Spring Session 还支持 JDBC, MongoDB 等方式
spring.session.store-type=redis
```

创建 Spring Session 配置类, 除了加注解 EnableRedisHttpSession 启用功能外, 主要是配置 RedisConnectionFactory 类型的 Bean, 另外需要注意的是 Spring Boot 2.x 使用的 Redis 客户端是 Luttce, 我这里也使用这个 Redis 客户端进行测试

```java
// Session 过期时间设置为 3600 秒
@Configuration
@EnableRedisHttpSession(maxInactiveIntervalInSeconds = 3600)
public class SessionConfig {
    
    @Bean
    public LettuceConnectionFactory connectionFactory() {
        RedisStandaloneConfiguration redisStandaloneConfiguration = new RedisStandaloneConfiguration("127.0.0.1", 6379);
        redisStandaloneConfiguration.setPassword(RedisPassword.of("lupengwei.4585"));
        LettuceClientConfiguration lettuceClientConfiguration = LettucePoolingClientConfiguration.builder()
                .commandTimeout(Duration.ofMillis(10000))
                .build();
        return new LettuceConnectionFactory(redisStandaloneConfiguration, lettuceClientConfiguration);
    }
}
```

创建测试类:

```java
@Slf4j
@RestController
public class SessionController {

    @GetMapping("/spring/session/data/set/test")
    public ResponseEntity register(HttpServletRequest request) {
        request.getSession().setAttribute("data", "lupengwei");
        Map<String, Object> map = new HashMap<>();
        map.put("code", "0");
        map.put("message", "success");
        return new ResponseEntity(map, HttpStatus.OK);
    }

    @GetMapping("/spring/session/data/get/test")
    public ResponseEntity getSessionMessage(HttpServletRequest request) {
        Map<String, Object> map = new HashMap<>();
        map.put("sessionId", request.getSession().getId());
        map.put("data",request.getSession().getAttribute("data")) ;
        return new ResponseEntity(map, HttpStatus.OK);
    }
}
```

启动项目, 报错, 找不到 GenericObjectPoolConfig 类, 添加以下依赖即可:

```xml
<dependency>
    <groupId>org.apache.commons</groupId>
    <artifactId>commons-pool2</artifactId>
</dependency>
```

再次启动项目后测试无误, 测试结果就不贴图, 参考资料:

* [SpringBoot 利用 Lettuce 获取 Redis Connection](https://www.cnblogs.com/koushr/p/9211801.html)
* [spring-session 使用文档](https://www.docs4dev.com/docs/zh/spring-session/2.1.2.RELEASE/reference/introduction.html#%E7%AE%80%E4%BB%8B)
* [Spring Boot 与 Redis 实现 Cache 以及 Session 共享](https://segmentfault.com/a/1190000012490895) - 讲了 Jedis, Luttce 两种集成方法

# 持久化到数据库中实现 Session 共享

当然也可以将共享的 Session 信息持久化到数据库中, 如 MySQL, 使用关系型数据库这种方式对数据库性能要求较高, 在高并发情况下会有性能瓶颈问题, 因此可以使用 MongoDB 这种非关系型数据库, 可以较好的解决高并发情况下性能瓶颈问题
