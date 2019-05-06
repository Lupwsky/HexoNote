---
title: Spring-源码阅读BeanFactory
date: 2019-04-14 20:53:10
categories: Spring
---

![xmlbeanfactory.png](https://cdn.nlark.com/yuque/0/2019/png/259000/1554948615958-78e34256-49aa-491c-a8f8-de0cc4fb48a1.png#align=left&display=inline&height=350&name=xmlbeanfactory.png&originHeight=652&originWidth=1388&size=42765&status=done&width=746)

<!-- more -->

# BeanFactory

BeanFactory 是一个接口, 他定义了 Spring 容器里面对 Bean 最基础的操作, 例如从获取 Bean, 判断一个 Bean 是否在容器中存在, 所有的容器要实现里面的接口, 源码如下:

```java
public interface BeanFactory {
    String FACTORY_BEAN_PREFIX = "&";
    Object getBean(String name) throws BeansException;
    <T> T getBean(String name, Class<T> requiredType) throws BeansException;
    Object getBean(String name, Object... args) throws BeansException;
    <T> T getBean(Class<T> requiredType) throws BeansException;
    <T> T getBean(Class<T> requiredType, Object... args) throws BeansException;

    <T> ObjectProvider<T> getBeanProvider(Class<T> requiredType);
    <T> ObjectProvider<T> getBeanProvider(ResolvableType requiredType);

    boolean containsBean(String name);
    boolean isSingleton(String name) throws NoSuchBeanDefinitionException;
    boolean isPrototype(String name) throws NoSuchBeanDefinitionException;
    boolean isTypeMatch(String name, ResolvableType typeToMatch) 
    throws NoSuchBeanDefinitionException;
    boolean isTypeMatch(String name, Class<?> typeToMatch) 
    throws NoSuchBeanDefinitionException;

    @Nullable
    Class<?> getType(String name) throws NoSuchBeanDefinitionException;
    String[] getAliases(String name);
}
```

# Scope

在这里了解 scope 是 BeanFactory 中有两个方法和他相关, 一个是 isPrototype(String name) 一个是 getBean(String name, Object... args) 方法, isPrototype(String name) 方法用于判断这个 bean 是否是一个 prototype 类型的 bean 如果不是一般就是 singleton 类型的, 可以通过 scope 来配置, scope 可以声明 bean 在容器中的存活空间, 有 singleton, prototype, request 和 session 类型

| 类型名 | 作用 |
| --- | --- |
| singleton | 一个容器只存在一个实例, 所有人共享这一个实例, 对容器而言是一个单例 |
| prototype | 每次都会生成一个新的实例, 容器只负责生成这个实例, 不负责管理此 bean |
| request  | 在 WEB 应用中才会去使用, 为每个 HTTP 请求都创建一个实例, 可以理解为 prototype 的一种特例 |
| session  | 在 WEB 应用中才会去使用在 WEB 应用中使用, 对于同一个会话, 这个 bean 只会创建一次 |

当配置的 bean 的 scope 为 prototype 的时候使用  getBean(String name, Object... args) 方法可以传入初始化参数生成一个新的 bean, 使用示例如下:

```xml
<bean id="userInfo" class="com.spring.study.beans.UserInfo" scope="prototype"/>
```

UserInfo.java:

```java
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class UserInfo {
    private String name;
    private String email;
}
```

Main.java:

```java
@Slf4j
public class Main {
    public static void main(String[] args) {
        XmlBeanFactory beanFactory = 
          new XmlBeanFactory(new ClassPathResource("/beans/UserBeans.xml"));
        UserInfo userInfo = 
          (UserInfo) beanFactory.getBean("userInfo", "lupengwei", "lupengwei@qq.com");
        log.info("{}", JSONObject.toJSONString(userInfo));
    }
}
```

参考资料:

* [Passing Arguments to getBean() Method in Spring](https://netjs.blogspot.com/2018/11/passing-arguments-to-getbean-method-spring.html)

# ObjectProvider

getBeanProvider 返回一个 ObjectProvider<T> 类型, 要了解这个方法的使用, 需要了解 ObjectProvider<T> 类, Spring 4.3 的时候提供了此类, 是 ObjectFactory 类的扩展, 提供了对依赖关系的注入机制进行了改进

ObjectProvider 提供的方法如下:

```java
// 如果指定类型的 bean 注册到容器中, 返回 bean 实例, 否则返回 null
T getIfAvailable() throws BeansException

// 如果指定类型的 bean 在容器中只有一个 bean, 返回 bean 实例, 如果不存在则返回 null
// 如果容器中有多个此类型的 bean, 抛出 NoUniqueBeanDefinitionException 异常
T getIfUnique() throws BeansException

// 返回指定类型的 bean, 如果容器中不存在, 抛出 NoSuchBeanDefinitionException 异常
// 如果容器中有多个此类型的 bean, 抛出 NoUniqueBeanDefinitionException 异常
T getObject() throws BeansException

// 同 T getObject(), 可传递创建 bean 时的属性参数
T getObject(Object... args) throws BeansException

// 5.0 之后又提供了如下方法
// 容器中有多个 bean, 返回指定类型 bean 实例的 迭代器 
Iterator<T> iterator();

// 转换成流, 便于流式处理
Stream<T> stream();

// 转换成有序流, 便于流式处理
Stream<T> orderedStream();

// 此外 getIfAvailable, getIfUnique 支持传递一个函数式接口, 当 bean 不存在时返回一个默认的值
T getIfAvailable(Supplier<T> defaultSupplier) throws BeansException;
T getIfUnique(Supplier<T> defaultSupplier) throws BeansException;
```

ObjectProvider 参考资料:

* [Spring-Programmatic Lookup of dependencies via ObjectProvider](https://www.logicbig.com/tutorials/spring-framework/spring-core/object-provider.html)
* [Spring-Java 8 functions support in ObjectProvider to provide callbacks](https://www.logicbig.com/tutorials/spring-framework/spring-core/object-provider-with-function-callbacks.html)
* [Spring-Java 8 Stream support in ObjectProvider to retrieve beans stream](https://www.logicbig.com/tutorials/spring-framework/spring-core/object-provider-bean-stream-retrieval.html)
* [Spring-Using ObjectProvider to Inject Narrower Scoped Bean](https://www.logicbig.com/tutorials/spring-framework/spring-core/object-provider-for-prototype-bean.html)
* [Spring Framework 4.3 注入机制的改进](https://www.huangyunkun.com/2016/03/06/spring-framework-4-3-new/)

上面的参考资料里面详细的介绍了 ObjectProvider 的使用, 测试代码就不贴了

```java
ObjectProvider<UserInfo> userInfoObjectProvider = beanFactory.getBeanProvider(UserInfo.class);
UserInfo userInfo = userInfoObjectProvider.getIfAvailable(UserInfo::new);
```