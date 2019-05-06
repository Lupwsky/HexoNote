---
title: Spring-源码阅读AutowireCapableBeanFactory
date: 2019-04-14 21:13:17
categories: Spring
---

![xmlbeanfactory.png](https://cdn.nlark.com/yuque/0/2019/png/259000/1554948615958-78e34256-49aa-491c-a8f8-de0cc4fb48a1.png#align=left&display=inline&height=350&name=xmlbeanfactory.png&originHeight=652&originWidth=1388&size=42765&status=done&width=746)

<!-- more -->

# AutowireCapableBeanFactory

AutowireCapableBeanFactory是对 BeanFactory 接口的扩展, 为 BeanFactory 提供了自动装配的功能, 和 ConfigurableBeanFactory 一样, 通常的应用程序也不会直接使用这个接口, 被设计为框架内部使用的, 例如 ApplicationContext 就没有实现这个接口, 但是可以从应用程序上下文中可以获取, 如 ApplicationContext.getAutowireCapableBeanFactory() 方法, 或者 BeanFactoryAware 接口, 获取 BenaFactory 实例后强转成 ConfigurableBeanFactory 类型即可, 源码如下:

```java
public interface AutowireCapableBeanFactory extends BeanFactory {

    // 非自动装配
    int AUTOWIRE_NO = 0;

    // 通过名称自动装配
    int AUTOWIRE_BY_NAME = 1;

    // 通过类型自动装配
    int AUTOWIRE_BY_TYPE = 2;

    // 通过构造器自动装配
    int AUTOWIRE_CONSTRUCTOR = 3;

    @Deprecated
    int AUTOWIRE_AUTODETECT = 4;

    // 初始化实例给定名称时约定的后缀, 会添加到全类名的后面, 例如: com.mypackage.MyClass.ORIGINAL
    String ORIGINAL_INSTANCE_SUFFIX = ".ORIGINAL";

    // 创建并执行 bean 的完整初始化操作, 包括执行 BeanPostProcessors
    <T> T createBean(Class<T> beanClass) throws BeansException;

    // 装配一个实例, 会进行初始化回调方法和后置处理方法
    void autowireBean(Object existingBean) throws BeansException;

    // 配置一个 bean, 包括装配 bean 的属性和值, 设置回调方法, 如 setName, setBeanFactory 等
    // 同时进行后置处理
    Object configureBean(Object existingBean, String beanName) throws BeansException;

    // autowireMode: 装配模式
    // dependencyCheck: 是否进行依赖检测
    Object createBean(Class<?> beanClass, int autowireMode, boolean dependencyCheck)
    throws BeansException;
    Object autowire(Class<?> beanClass, int autowireMode, boolean dependencyCheck)
    throws BeansException;
    void autowireBeanProperties(Object existingBean, int autowireMode, boolean dependencyCheck)
            throws BeansException;

    // 将指定 bean 的 BeanDefinition 应用到 existingBean
    void applyBeanPropertyValues(Object existingBean, String beanName)
    throws BeansException;
  
    // 通过指定 bean 来初始化 existingBean, existingBean 的 setBeanName 和 setBeanFactory
    // 等都是和从指定 bean 中的是一样的
    Object initializeBean(Object existingBean, String beanName)
    throws BeansException;

    // 将指定 bean 的 BeanPostProcessors 应用到 existingBean 上面
    // 并调用 postProcessAfterInitialization 方法
    Object applyBeanPostProcessorsBeforeInitialization(Object existingBean, String beanName)
            throws BeansException;

    // 将指定 bean 的 BeanPostProcessors 应用到 existingBean 上面
    // 并调用 postProcessAfterInitialization 方法
    Object applyBeanPostProcessorsAfterInitialization(Object existingBean, String beanName)
            throws BeansException;

    // 销毁 bean 实例, 会调用 DisposableBean 和 DestructionAwareBeanPostProcessor 中的方法
    void destroyBean(Object existingBean);

    // 解析匹配类型的 bean 实例, 是 getBean 的变体, 只不过该方法返回 bean 的名称信息
    <T> NamedBeanHolder<T> resolveNamedBean(Class<T> requiredType)
    throws BeansException;
  
    Object resolveBeanByName(String name, DependencyDescriptor descriptor)
    throws BeansException;

    Object resolveDependency(DependencyDescriptor descriptor,
                           String requestingBeanName) throws BeansException;
    Object resolveDependency(DependencyDescriptor descriptor,
                           String requestingBeanName,
                                                     Set<String> autowiredBeanNames,
                           TypeConverter typeConverter) throws BeansException;

}
```