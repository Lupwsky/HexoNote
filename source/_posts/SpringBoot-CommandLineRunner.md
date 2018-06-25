---
title: SpringBoot-CommandLineRunner
date: 2018-06-22 00:35:53
categories: Spring
---

CommandLineRunner 是 SpringBoot 提供的一个接口类，可以在项目服务启动的时候进行初始化操作或者加载其他的一些数据，其定义如下。

```java
public interface CommandLineRunner {
    void run(String... var1) throws Exception;
}
```

# CommandLineRunner 基础使用

下面的例子直接在 Application 类中实现 CommandLineRunner 接口类，在 SpringBoot 项目启动后在控制台输出输出日志数据。

```java
@SpringBootApplication
public class Application implements CommandLineRunner {
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }

    @Override
    public void run(String... strings) throws Exception {
        System.out.println("测试");
    }
}
```

<!-- more -->

控制台输出：

```text
Tomcat started on port(s): 8001 (http)
测试
Started Application in 2.405 seconds (JVM running for 3.827)
```

可以单独定义类，实现 CommandLineRunner 接口类，对于多个实现了CommandLineRunner 接口类的类， 可以使用 @Order(value = ?) 注解的方式定义先后执行的顺序。

```java
@SpringBootApplication
@Order(value = 3)
public class Application implements CommandLineRunner {
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }

    @Override
    public void run(String... strings) throws Exception {
        System.out.println("测试3");
    }
}

@Component
@Order(value = 1)
public class TestCommandLineRunner implements CommandLineRunner {
    @Override
    public void run(String... strings) throws Exception {
        System.out.println("测试1");
    }
}

// @Commonet，@Repository，@Server，@Configuration 都可以使 CommandLineRunner 生效
@Component
@Order(value = 2)
public class TestCommandLineRunner2 implements CommandLineRunner {
    @Override
    public void run(String... strings) throws Exception {
        System.out.println("测试2");
    }
}
```

控制台输出如下：

```text
Tomcat started on port(s): 8001 (http)
测试1
测试2
测试3
Started Application in 2.132 seconds (JVM running for 2.743)
```