---
title: SpringBoot-Bean的初始之前操作和销毁之前操作
date: 2018-06-22 01:08:25
categories: Spring
---

本篇学习笔记仅记录 Bean 的初始之前操作和销毁之前操作的实现方式的基础使用方法。

# @PostConstruct 和 @PreDestroy

JDK 1.5 引入了 @PostConstruct 和 @PreDestroy 这两个作用于 Servlet 生命周期的注解，实现 Bean 初始化之前和销毁之前的自定义操作，这两个注解使用时有以下几个注意的地方。

* 只有一个方法可以使用此注释进行注解；
* 被注解方法不得有任何参数；
* 被注解方法返回值为 void；
* 被注解方法不得抛出已检查异常；
* 被注解方法需是非静态方法；
* 此方法只会被执行一次且在构造方法之后执行。

<!-- more -->

```java
public class UserBean {

    @PostConstruct
    public void init() {
        System.out.println("init");
    }

    public void getUserInfo() {
        System.out.println("getUserInfo");
    }

    @PreDestroy
    public void destroy() {
        System.out.println("destroy");
    }
}
```

# InitializingBean 和 DisposableBean

Spring 框架提供的 InitializingBean 和 DisposableBean 也可以实现 Bean 的初始之前操作和销毁之前操作。

```java
public class UserBean implements InitializingBean, DisposableBean {
    public void getUserInfo() {
        System.out.println("getUserInfo");
    }

    @Override
    public void destroy() throws Exception {
        System.out.println("destroy");
    }

    @Override
    public void afterPropertiesSet() throws Exception {
        System.out.println("init");
    }
}
```

当然 Spring 框架也提供使用 xml 文件配置的方式，配置 init-method 和  destory-method 方法，该方式的使用方法之后再记录。