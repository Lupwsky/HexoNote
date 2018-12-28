---
title: Spring-XML方式配置Bean的
date: 2018-12-18 22:51:49
tags:
---

最近开始复习 Spring 的相关知识, 为看 Spring 的源码做好准备, 熟悉 Spring 的人都知道 Spring 有三种方式可以配置 Bean, 如下:

* 基于 XML 的配置方式
* 基于注解的配置方式
* 基于 Java 类的配置方式

<!-- more -->

# XML 配置 Bean 的相关属性

Spring 为 XML 配置 Bean 时定义了一些属性, 下面来一一了解这些属性的作用和用法

  属性名   |          描述
---------- | ----------------------
id 和 name | bean 在容器中的的唯一标识符

# id 和 name 的区别

* id 和 name 都是 Spring 容器中 bean 的唯一标识符
* bean 的名称可以有多个, 如 name="beanName1, beanName2, beanName3", 但是 id 只能指定一个
* 如果 id 没有配置, 默认使用 name 的第一个作为 id
* 如果 id 和 name 都没有配置, Spring 默认使用类的全类名作为 id 和 name, 如 com.lupw.spring.domain.UserInfo

对于多个同类型的 bean, 如果 id 和 name 都没有配置, Spring 会按照其出现的次序, 分别指定 id 和 name, 如 com.lupw.spring.domain.UserInfo, com.lupw.spring.domain.UserInfo#1, com.lupw.spring.domain.UserInfo#2, 示例如下:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd">

    <bean class="com.lupw.spring.domain.UserInfo">
        <property name="name" value="卢鹏伟"/>
        <property name="sex" value="男"/>
    </bean>

    <bean class="com.lupw.spring.domain.UserInfo">
        <property name="name" value="卢鹏伟1"/>
        <property name="sex" value="男"/>
    </bean>

    <bean class="com.lupw.spring.domain.UserInfo">
        <property name="name" value="卢鹏伟2"/>
        <property name="sex" value="男"/>
    </bean>
</beans>
```

使用 Spring 分配好的 id 获取, 代码如下:

```java
Resource resource = new FileSystemResource("/Users/lupengwei/IDEA/MSpringMvc/src/main/resources/beans/bean.xml");
XmlBeanFactory xmlBeanFactory = new XmlBeanFactory(resource);
UserInfo userInfo = (UserInfo) xmlBeanFactory.getBean("com.lupw.spring.domain.UserInfo");
UserInfo userInfo1 = (UserInfo) xmlBeanFactory.getBean("com.lupw.spring.domain.UserInfo#1");
UserInfo userInfo2 = (UserInfo) xmlBeanFactory.getBean("com.lupw.spring.domain.UserInfo#2");
PrintUtil.print(JSONObject.toJSONString(userInfo));
PrintUtil.print(JSONObject.toJSONString(userInfo1));
PrintUtil.print(JSONObject.toJSONString(userInfo2));
```

输出结果如下:

```text
{"name":"卢鹏伟","sex":"男"}
{"name":"卢鹏伟1","sex":"男"}
{"name":"卢鹏伟2","sex":"男"}
```