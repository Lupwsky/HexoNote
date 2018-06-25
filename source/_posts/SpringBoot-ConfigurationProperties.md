---
title: SpringBoot-ConfigurationProperties
date: 2018-06-23 13:16:42
categories: Spring
---

Configuration 注解可以用来定义一个配置类，需要为每一个属性声明绑定的属性的字段，先简单了解下是 Configuration 注解是如何使用的。

# Configuration

Configuration 注解用于定义配置类，可以用来他取代 xml 配置，配置的属性值通常放定义在 properties 文件中，通过绑定字段，在创建相应的 Bean 是会自动的将值填入绑定的类的成员属性中，例如将下面的 ServerProperties 中的 port 值以这种方法将值填入的例子。

定义一个将用于配置用的 ServerPropertie 类：

```java
@Data
public class ServerProperties {
    private int port;
}
```

先在配置文件中定义一个属性字段

```ini
def.server.port=6000
```

<!-- more -->

接着在 ServerProperties 类中添加 @Configuration 注解，并为相应的成员属性绑定 (使用 @Value 注解) 对应的属性值：

```java
@Configuration
@Data
public class ServerProperties {
    @Value("def.server.port")
    private int port;
}
```

在创建 Bean 的使用，会去 classpath 目录下读取所有的配置文件，寻找在配置文件中定义好的 def.server.port 属性值，然后填入 ServerProperties 类实例的 prot 字段中，为了便于框架准确的定位属性文件快速的找到对应的属性值，可以使用 @PropertySource 注解指定配置文件路径，如下所示：

```java
// 设置配置文件， value 接收 String[] 类型的值，多个路径 value = {"classpath:xxx", "classpath:xxx"} 的格式设置
@PropertySource(value = "classpath:application.properties")
@Configuration
@Data
public class ServerProperties {
    @Value("def.server.port")
    private int port;
}
```

接着就可以开始定义 Bean 和使用定义好的 Bean 了，如下：

```java
// 定义
@Configuration
public class ServerConfiguation {
    @Bean
    public ServerProperties getServerProperties() {
        return ServerProperties();
    }
}

// 使用
@Controller
public class CommonController {
    private final ServerProperties serverProperties;

    @Autowired
    public CommonController(ServerProperties srverProperties) {
        this.srverProperties = srverProperties;
        log.info(srverProperties.getPort());
    }
}
```

Configuration 注解的一些使用总结：

* 配置类不能是 final 类型，也不能是匿名类，且嵌套的 @configuration 注解的配置类必须是静态类
* @Configuation 等价于 xml 配置文件中的 `<Beans></Beans>`
* @Bean 等价于 xml 配置文件中的 `<Bean></Bean>`
* @ComponentScan 等价于 xml 配置文件中的 `<context:component-scan base-package="com.dxz.demo"/>`

# ConfiggurationPropertoes

ConfiggurationPropertoes 注解可以将配置文件的属性映射成一个 POJO 类，只要属性名一致就能自动注入，相比 Configuration 更加的简单易用 (需要配合 @EnableConfigurationProperties 使用)。上面的例子使用 注解时，ServerProperties 类更改如下，首先是不再需要再类上面使用 @Configuration 注解了，而是使用 @ConfigurationProperties，并指定一个属性前缀，同时类的成员属性上面也不需要 @Value 注解来绑定属性名了，只需要前缀一致，属性名和类的成员属性名一致即可由自动注入，修改后的代码如下：

```java
// 设置配置文件， value 接收 String[] 类型的值，多个路径 value = {"classpath:xxx", "classpath:xxx"} 的格式设置
@PropertySource(value = "classpath:application.properties")
// 属性值匹配的前缀
@ConfigurationProperties(prefix="def.server")
@Data
public class ServerProperties {
    private int port;
}
```

接着就可以开始使用 Bean 了，在需要使用该 Bean 的类上面添加 @EnableConfigurationProperties 注解，如下所示：

```java
@EnableConfigurationProperties(value = {ServerProperties.class})
public class CommonController {
    private final ServerProperties serverProperties;

    @Autowired
    public CommonController(ServerProperties srverProperties) {
        this.srverProperties = srverProperties;
        log.info(srverProperties.getPort());
    }
}
```