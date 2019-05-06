---
title: Spring-源码阅读SingletonBeanRegistry
date: 2019-04-14 21:11:24
categories:  Spring
---

![xmlbeanfactory.png](https://cdn.nlark.com/yuque/0/2019/png/259000/1554948615958-78e34256-49aa-491c-a8f8-de0cc4fb48a1.png#align=left&display=inline&height=350&name=xmlbeanfactory.png&originHeight=652&originWidth=1388&size=42765&status=done&width=746)

<!-- more -->

# SingletonBeanRegistry

SingletonBeanRegistry 可以被任意的容器继承和实现, SingletonBeanRegistry 定义了对单例 bean 操作的方法, 源码如下:

```java
public interface SingletonBeanRegistry {
    // 将某个实例注册以单例模式注册到容器
    // 通过此方法注册的 bean 不会调用 InitializingBean 的 afterPropertiesSet
    // 和 DisposableBean 的 destroy 方法
    void registerSingleton(String beanName, Object singletonObject);

    // 获取 bean
    Object getSingleton(String beanName);

    // 判断是否包含 bean
    boolean containsSingleton(String beanName);

    // 获取所有单实例的 bean 名称
    String[] getSingletonNames();

    // 获取所有单实例的数量
    int getSingletonCount();

    Object getSingletonMutex();
}
```

# registerSingleton

将手动创建的对象以单例模式注册到容器中, 创建 bean 如下:

```xml
<bean id="userInfo" class="com.spring.study.beans.UserInfo">
    <constructor-arg name="name" value="lupengwei" />
    <constructor-arg name="email" value="lupengwei@163.com"/>
</bean>
```

测试代码如下:

```java
XmlBeanFactory beanFactory =
  new XmlBeanFactory(new ClassPathResource("/beans/UserBeans.xml"));
log.info("{}", Arrays.toString(beanFactory.getSingletonNames()));

// 创建一个新的对象, 注册到容器中
UserInfoA userInfoA = new UserInfoA();
UserInfoA userInfoA = new UserInfoA();
beanFactory.registerSingleton("userInfoA", userInfoA);
log.info("{}", Arrays.toString(beanFactory.getSingletonNames()));
```

输出结果如下:

```text
[userInfo]
[userInfo, userInfoA]
```

# getSingletonMutex

暂不清楚这个方法的实际作用, 使用 registerSingleton 方法的测试代码测试, 发现 Object 是一个 ConcurrentHashMap 类型, 里面存放的是当前已注册到容器里面的 bean, 如下图:

![WX20190411-232929@2x.png](https://cdn.nlark.com/yuque/0/2019/png/259000/1554996595745-21142277-77f9-4e24-af4d-742101849be6.png#align=left&display=inline&height=43&name=WX20190411-232929%402x.png&originHeight=116&originWidth=2030&size=49369&status=done&width=746)