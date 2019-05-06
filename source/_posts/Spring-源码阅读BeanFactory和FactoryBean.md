---
title: Spring-源码阅读BeanFactory和FactoryBean
date: 2019-04-05 10:43:48
categories: Spring
---

# BeanFactory 和 FactoryBean

* BeanFactory 是一个接口, 他定义了 Spring 容器里面对 Bean 最基础的操作, 例如从获取 Bean, 判断一个 Bean 是否在容器中存在, 所有的容器要实现这个接口
* FactoryBean 是一个接口, 普通的 Bean 如果实现这个接口可以变成一个特殊的 Bean (工厂 Bean, 可以让容器管理), 他可以为普通的 Bean 提供了一个简单工厂模式, 通过配置创建不同的对象

# FactoryBean 使用简单示例

创建一个 BeanFactoryBean 类实现 FactoryBean 的三个接口, 分别如下:

```java
@Data
@Builder
@AllArgsConstructor
@NoArgsConstructor
public class UserInfoA {
    private String userInfoA;
}
```

<!-- more -->

```java
@Data
@Builder
@AllArgsConstructor
@NoArgsConstructor
@EqualsAndHashCode(callSuper = false)
public class UserInfoB {
    private String userInfoB;
}
```

```java
@Data
public class BeanFactoryBean implements FactoryBean {

    private String userInfoType;

    // 返回生成的对象
    @Override
    public Object getObject() throws Exception {
        switch (userInfoType) {
            case "userInfoA": return UserInfoA.builder().userInfoA("A").build();
            case "userInfoB": return UserInfoB.builder().userInfoB("B").build();
            default: throw new RuntimeException("不存在此类型的 Bean");
        }
    }

    // 返回生成对象的类型
    @Override
    public Class<?> getObjectType() {
        switch (userInfoType) {
            case "userInfoA": return UserInfoA.class;
            case "userInfoB": return UserInfoB.class;
            default: throw new RuntimeException("不存在此类型的 Bean 的类型");
        }
    }

    // 生成的对象是否是单例的, true = 是, false = 否
    @Override
    public boolean isSingleton() {
        return true;
    }
}
```

在 XML 文件中根据配置从工厂 BeanFactoryBean 中获取不同类型的 Bean 实例, XML 配置如下:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd">
    <!--
        通过配置工厂 Bean 的 userInfoType 属性来生成不同 Bean, 这两个 Bean 都是单实例的
    -->
    <bean id="userInfoA" class="com.spring.study.beans.BeanFactoryBean">
        <property name="userInfoType" value="userInfoA"/>
    </bean>

    <bean id="userInfoB" class="com.spring.study.beans.BeanFactoryBean">
        <property name="userInfoType" value="userInfoB"/>
    </bean>
</beans>
```

在代码中获取和使用 Bean, 示例代码如下:

```java
public static void main(String[] args) {
    DefineClassPathXmlApplicationContext applicationContext = new DefineClassPathXmlApplicationContext("beans/UserBeans.xml");
    UserInfo userInfo = applicationContext.getBean("userInfo", UserInfo.class);
    log.info("name = {}, email = {}", userInfo.getName(), userInfo.getEmail());

    UserInfoA userInfoA = applicationContext.getBean("userInfoA", UserInfoA.class);
    log.info("type = {}", userInfoA.getUserInfoA());

    UserInfoB userInfoB = applicationContext.getBean("userInfoB", UserInfoB.class);
    log.info("type = {}", userInfoB.getUserInfoB());
}
```

如果需要获取这个工厂 Bean 本身, 在他能生产的 Bean 的名字前加上 "&" 即可, 示例代码如下:

```java
// "&userInfoA" 或者 "&userInfoB" 可以获取到 FactoryBean 本身这个 Bean
BeanFactoryBean beanFactoryBean = applicationContext.getBean("&userInfoA", BeanFactoryBean.class);
try {
    // 通过工厂 Bean 创建一个 UserInfoA 的实例
    beanFactoryBean.setUserInfoType("userInfoA");
    UserInfoA userInfoA1 = (UserInfoA) beanFactoryBean.getObject();
    if (userInfoA1 != null) {
        log.info("type = {}", userInfoA1.getUserInfoA());
    }
} catch (Exception e) {
    e.printStackTrace();
}
```