---
title: Spring-SpringBoot项目在配置文件中配置复杂类型属性值
date: 2019-01-26 14:23:29
categories:  Spring
---

在项目中, 经常在 application.properties 配置文件中配置一些配置项, 然后在使用的时候用 @Value(value = "${propertiesName}") 来注入, 但是这只是简单 String 类型,  下面记录下对于 Map, List 等这些复杂类型的配置方法

# Map<String, String>

定义属性值:

``` ini
# 注意在 Java 配置类中的峰驼式风格属性名不能使用点分隔, 如 spring.datasource.crm.properties 在这个例子中就会报错
spring.datasource.crm-properties.url=HTM0
spring.datasource.crm-properties.username=NAME0
```

对应的配置类:

```java
 @Data
 @Component
 @ConfigurationProperties(prefix = "spring.datasource")
 public class MultiProperties {
     // 和配置文件中的属性名名 crm-properties 对应
     private Map<String, String> crmProperties;
 }
```

<!-- more -->

使用方法:

```java
// 需要注入 MultiProperties 类型 Bean 的类上添加 @EnableConfigurationPropertie 注解
@EnableConfigurationProperties(value = MultiProperties.class)

// 注入
@Autowired
private MultiProperties multiProperties;

// 获取注入的属性值
Map<String, String> map = multiProperties.getCrmProperties();
log.info("key = {}, value = {}", map.get("url"), map.get("username"));
```

# Map<String, Object>

定义属性值:

```ini
spring.datasource.crm-properties[key0].url=HTM0
spring.datasource.crm-properties[key0].username=NAME0
spring.datasource.crm-properties[key1].url=HTM1
spring.datasource.crm-properties[key1].username=NAME1
spring.datasource.crm-properties[key2].url=HTM2
spring.datasource.crm-properties[key2].username=NAME2
```

对应的配置类:

```java
@Data
@Component
@ConfigurationProperties(prefix = "spring.datasource")
public class MultiProperties {
    private Map<String, DataSourceProperties> crmProperties;
}

@Data
public class DataSourceProperties {
    private String url;
    private String username;
}
```

使用方法:

```java
// 需要注入 MultiProperties 类型 Bean 的类上添加 @EnableConfigurationPropertie 注解
@EnableConfigurationProperties(value = MultiProperties.class)

// 注入
@Autowired
private MultiProperties multiProperties;

// 获取注入的属性值
Map<String, DataSourceProperties> maps = multiProperties.getCrmProperties();
DataSourceProperties dataSourceProperties0 = maps.get(key0);
DataSourceProperties dataSourceProperties1 = maps.get(key1);
DataSourceProperties dataSourceProperties2 = maps.get(key2);
```

# List<Map<String, String>>

定义属性值:

```ini
spring.datasource.crm-properties[0].url=HTM0
spring.datasource.crm-properties[0].username=NAME0
spring.datasource.crm-properties[1].url=HTM1
spring.datasource.crm-properties[1].username=NAME1
spring.datasource.crm-properties[2].url=HTM2
spring.datasource.crm-properties[2].username=NAME2
```

对应的配置类:

```java
@Data
@Component
@ConfigurationProperties(prefix = "spring.datasource")
public class MultiProperties {
    private List<Map<String, String>> crmProperties;
}
```

使用方法:

```java
// 需要注入 MultiProperties 类型 Bean 的类上添加 @EnableConfigurationPropertie 注解
@EnableConfigurationProperties(value = MultiProperties.class)

// 注入
@Autowired
private MultiProperties multiProperties;

// 获取注入的属性值
List<Map<String, String>> list = multiProperties.getCrmProperties();
list.forEach(map - > {
    log.info("key = {}, value = {}", map.get("url"), map.get("username"));
});
```

# Object

定义属性值:

```ini
spring.datasource.crm-properties.url=HTM0
spring.datasource.crm-properties.username=NAME0
```

对应的配置类:

```java
@Data
@Component
@ConfigurationProperties(prefix = "spring.datasource.crm-properties")
public class DataSourceProperties {
    private String url;
    private String username;
}
```

使用方法:

```java
// 需要注入 MultiProperties 类型 Bean 的类上添加 @EnableConfigurationPropertie 注解
@EnableConfigurationProperties(value = MultiProperties.class)

// 注入
@Autowired
private MultiProperties multiProperties;

// 获取注入的属性值
log.info("key = {}, value = {}", dataSourceProperties.getUrl(), dataSourceProperties.getUsername());
```

# List&lt;Object>

定义属性值:

```ini
spring.datasource.crm-properties[0].url=HTM0
spring.datasource.crm-properties[0].username=NAME0
spring.datasource.crm-properties[1].url=HTM1
spring.datasource.crm-properties[1].username=NAME1
spring.datasource.crm-properties[2].url=HTM2
spring.datasource.crm-properties[2].username=NAME2
```

对应的配置类:

```java
@Data
@Component
@ConfigurationProperties(prefix = "spring.datasource")
public class MultiProperties {
    private List<DataSourceProperties> crmProperties;
}

@Data
public class DataSourceProperties {
    private String url;
    private String username;
}
```

使用方法:

```java
// 需要注入 MultiProperties 类型 Bean 的类上添加 @EnableConfigurationPropertie 注解
@EnableConfigurationProperties(value = MultiProperties.class)

// 注入
@Autowired
private MultiProperties multiProperties;

// 获取注入的属性值
List<DataSourceProperties> list = multiProperties.getCrmProperties();
    list.forEach(dataSourceProperties -> {
    log.info("key = {}, value = {}", dataSourceProperties.getUrl(), dataSourceProperties.getUsername());
});
```