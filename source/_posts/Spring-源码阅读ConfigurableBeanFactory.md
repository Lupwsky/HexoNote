---
title: Spring-源码阅读ConfigurableBeanFactory
date: 2019-04-14 21:07:29
categories: Spring
---

![xmlbeanfactory.png](https://cdn.nlark.com/yuque/0/2019/png/259000/1554948615958-78e34256-49aa-491c-a8f8-de0cc4fb48a1.png#align=left&display=inline&height=350&name=xmlbeanfactory.png&originHeight=652&originWidth=1388&size=42765&status=done&width=746)

<!-- more -->

# ConfigurableBeanFactory

ConfigurableBeanFactory 是 BeanFactory 的扩展, 里面定义的接口比较多, 主要用于配置 BeanFactory, 这个接口里面定义的方法通常都是供框架内部使用的, 不适用于通常的应用程序, 源码如下:

```java
public interface ConfigurableBeanFactory extends HierarchicalBeanFactory,
                                                 SingletonBeanRegistry {
    String SCOPE_SINGLETON = "singleton";
    String SCOPE_PROTOTYPE = "prototype";

    void setParentBeanFactory(BeanFactory parentBeanFactory)
      throws IllegalStateException;

    // 两个设置 ClassLoader 的方法, 通常使用 setBeanClassLoader
    void setBeanClassLoader(@Nullable ClassLoader beanClassLoader);
    ClassLoader getBeanClassLoader();
    void setTempClassLoader(@Nullable ClassLoader tempClassLoader);
    ClassLoader getTempClassLoader();

    // 是否缓存 bean 的源数据, 默认为 true
    void setCacheBeanMetadata(boolean cacheBeanMetadata);
    boolean isCacheBeanMetadata();

    // 设置 SPEL 表达式解析器
    void setBeanExpressionResolver(@Nullable BeanExpressionResolver resolver);
    BeanExpressionResolver getBeanExpressionResolver();

    void setConversionService(@Nullable ConversionService conversionService);
    ConversionService getConversionService();

    // 设置属性编辑器
    void addPropertyEditorRegistrar(PropertyEditorRegistrar registrar);
  
    // 注册自定义的属性编辑器
    void registerCustomEditor(Class<?> requiredType,
                              Class<? extends PropertyEditor> propertyEditorClass);

    // 复制属性编辑器到当前属性编辑器
    void copyRegisteredEditorsTo(PropertyEditorRegistry registry);

    // 设置类型转换器
    void setTypeConverter(TypeConverter typeConverter);
    TypeConverter getTypeConverter();

    void addEmbeddedValueResolver(StringValueResolver valueResolver);
    boolean hasEmbeddedValueResolver();
    String resolveEmbeddedValue(String value);

    // 设置 bean 后置处理器
    void addBeanPost Processor(BeanPostProcessor beanPostProcessor);
    int getBeanPostProcessorCount();

    // 注册 scope
    void registerScope(String scopeName, Scope scope);
    String[] getRegisteredScopeNames();
    Scope getRegisteredScope(String scopeName);

    AccessControlContext getAccessControlContext();

    void copyConfigurationFrom(ConfigurableBeanFactory otherFactory);

    // 为 bean 设置添加别名
    void registerAlias(String beanName, String alias)
      throws BeanDefinitionStoreException;
    void resolveAliases(StringValueResolver valueResolver);

    BeanDefinition getMergedBeanDefinition(String beanName)
      throws NoSuchBeanDefinitionException;

    boolean isFactoryBean(String name) throws NoSuchBeanDefinitionException;

    void setCurrentlyInCreation(String beanName, boolean inCreation);
    boolean isCurrentlyInCreation(String beanName);

    void registerDependentBean(String beanName, String dependentBeanName);
    String[] getDependentBeans(String beanName);
    String[] getDependenciesForBean(String beanName);

    void destroyBean(String beanName, Object beanInstance);
    void destroyScopedBean(String beanName);
    void destroySingletons();
}
```