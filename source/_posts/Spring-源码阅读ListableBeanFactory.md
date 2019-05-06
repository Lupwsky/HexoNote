---
title: Spring-源码阅读ListableBeanFactory
date: 2019-04-14 20:57:25
categories: Spring
---

![xmlbeanfactory.png](https://cdn.nlark.com/yuque/0/2019/png/259000/1554948615958-78e34256-49aa-491c-a8f8-de0cc4fb48a1.png#align=left&display=inline&height=350&name=xmlbeanfactory.png&originHeight=652&originWidth=1388&size=42765&status=done&width=746)

<!-- more -->

# ListableBeanFactory

ListableBeanFactory 是对 BeanFactory 接口的扩展,和 BeanFactory 每次获取单个 bean 不同, ListableBeanFactory 可以一次性枚举出符合条件的 bean, 如果这个 BeanFactory 继承了 HierarchicalBeanFactory, ListableBeanFactory  接口面列出的 bean 不会列出祖先 BeanFactory 中的 bean,  并且会忽略通过其他方式注册的任何单例 bean, 如 ConfigurableBeanFactory 的 registerSingleton 方法, 但 getBeanNamesOfType 和 getBeansOfType 除外, 源码如下:

```java
public interface ListableBeanFactory extends BeanFactory {
    // 当前容器是否包含指定名称 bean
    boolean containsBeanDefinition(String beanName);
  
    // 当前容器注册的 bean 的数量
    int getBeanDefinitionCount();
  
    // 返回容器中注册的全部 bean 的名称
    String[] getBeanDefinitionNames();
  
    // 获取指定类型 (包括其子类) bean 的名称
    // includeNonSingletons: 是否返回单例模式的 bean
    // allowEagerInit: 是否允许预加载对象，以获取对象属性, emmmm..., 没有弄懂这个参数的含义
    //                 网上说是是否懒加载, 给 bean 设置 @Lazy 的值为 true 和 false 都能返回
    String[] getBeanNamesForType(ResolvableType type);
    String[] getBeanNamesForType(@Nullable Class<?> type);
    String[] getBeanNamesForType(@Nullable Class<?> type,
                               boolean includeNonSingletons,
                               boolean allowEagerInit);
  
    // 获取指定类型的 bean, 符合条件的 bean 存放在一个 map 中
    // 这个 map 的 key 值为 bean 的名称, value 为对应的实例
    <T> Map<String, T> getBeansOfType(@Nullable Class<T> type) throws BeansException;
    <T> Map<String, T> getBeansOfType(@Nullable Class<T> type,
                                    boolean includeNonSingletons,
                                    boolean allowEagerInit) throws BeansException;

    // 获取使用了指定注解的 bean 的名称
    String[] getBeanNamesForAnnotation(Class<? extends Annotation> annotationType);
  
    // 获取使用了指定注解的 bean, 符合条件的 bean 存放在一个 map 中
    Map<String, Object> getBeansWithAnnotation(Class<? extends Annotation> annotationType)
    throws BeansException;

    // 或者指定 bean 的注解信息
    @Nullable
    <A extends Annotation> A findAnnotationOnBean(String beanName, Class<A> annotationType)
            throws NoSuchBeanDefinitionException;

}
```

在官方的文档中提示了一个注意的地方, getBeanDefinitionCount 和 containsBeanDefinition 方法不是为频繁调用而设计的, 频繁的调用需要考虑到性能问题

# containsBeanDefinition

创建 UserBeans.java 文件, 使用注解方式创建一个 bean, 示例代码如下:

```java
@Component
public class UserInfoBeans {
    @Bean
    public UserInfo getUserInfo() {
        return UserInfo.builder().build();
    }
}
```

测试示例:

```java
// 输出 false
log.info("{}", applicationContext.containsBeanDefinition("userInfo"));

// 输出 true
log.info("{}", applicationContext.containsBeanDefinition("getUserInfo"));
```

这里要注意 bean 的配置, 如果不指定 bean 的名称, bean 的名称就是方法的名称, 所以上面的代码通过名称 userInfo 判断 bena 是否注册到容器中时返回的时 false

# getBeanDefinitionNames

返回容器中注册的全部 bean 的名字

```java
@Component
public class UserInfoBeans {
    @Bean
    public UserInfo userInfo() {
        return UserInfo.builder().build();
    }

    @Bean
    public UserInfoA userInfoA() {
        return UserInfoA.builder().build();
    }

    @Bean
    public UserInfoB userInfoB() {
        return UserInfoB.builder().build();
    }
}
```

测试示例:

```java
log.info("{}", Arrays.toString(applicationContext.getBeanDefinitionNames()));
```

输出结果示例:

```text
[org.springframework.context.annotation.internalConfigurationAnnotationProcessor,
org.springframework.context.annotation.internalAutowiredAnnotationProcessor,
org.springframework.context.annotation.internalCommonAnnotationProcessor,
org.springframework.context.event.internalEventListenerProcessor,
org.springframework.context.event.internalEventListenerFactory,
application,
org.springframework.boot.autoconfigure.internalCachingMetadataReaderFactory,
userInfoBeans,
beanTestController,
testController,
userInfo,
userInfoA,
userInfoB, ...]
```

# getBeanNamesForType

示例代码如下:

```java
ResolvableType resolvableType = ResolvableType.forClass(UserInfo.class);
log.info("{}", Arrays.toString(applicationContext.getBeanNamesForType(resolvableType)));
```

输出结果如下:

```text
[userInfoB]
```

# getBeansOfType

示例代码如下:

```java
Map<String, UserInfo> userInfoMap = applicationContext.getBeansOfType(UserInfo.class);
UserInfo userInfo = userInfoMap.get("userInfo");
userInfo.setName("lpw");
userInfo.setEmail("lpw@qq.com");
log.info("{}", JSONObject.toJSONString(userInfo));
```

输出结果如下:

```text
{"email":"lpw@qq.com","name":"lpw"}
```

# getBeansWithAnnotation

创建一个定义的注解, 示例代码如下:

```java
public @interface MyAnnotation {
    String value() default "";
}
```

在定义 bean 的时候使用这个注解, 示例代码如下:

```java
@Bean
@MyAnnotation(value = "hello")
public UserInfo userInfo() {
    return UserInfo.builder().build();
}
```

测试代码如下:

```java
Map<String, Object> map = applicationContext.getBeansWithAnnotation(MyAnnotation.class);
UserInfo info = (UserInfo) map.get("userInfo");
info.setName("lpw");
info.setName("lpw@qq.com");
log.info("{}", JSONObject.toJSONString(info));
```

# findAnnotationOnBean

在定义 bean 的时候使用这个注解, 示例代码如下:

```java
@Bean
@MyAnnotation(value = "hello")
public UserInfo userInfo() {
    return UserInfo.builder().build();
}
```

获取 bean 名称为 userInfo 的 bean 使用的注解信息, 并获取 value 值, 示例代码如下:

```java
MyAnnotation myAnnotation =
  applicationContext.findAnnotationOnBean("userInfo", MyAnnotation.class);
log.info("{}", myAnnotation.value());
```

输出结果如下:

```text
hello
```