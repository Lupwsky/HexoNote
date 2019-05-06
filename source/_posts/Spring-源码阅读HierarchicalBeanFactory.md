---
title: Spring-源码阅读HierarchicalBeanFactory
date: 2019-04-14 21:05:52
categories: Spring
---

![xmlbeanfactory.png](https://cdn.nlark.com/yuque/0/2019/png/259000/1554948615958-78e34256-49aa-491c-a8f8-de0cc4fb48a1.png#align=left&display=inline&height=350&name=xmlbeanfactory.png&originHeight=652&originWidth=1388&size=42765&status=done&width=746)

<!-- more -->

# HierarchicalBeanFactory

HierarchicalBeanFactory 是对 BeanFactory 接口的扩展, 提供了对父容器的访问功能 (对父容器的设置功能在 ConfigurableBeanFactory 中定义), 源码如下:

```java
public interface HierarchicalBeanFactory extends BeanFactory {
    // 返回父 BeanFactory
    @Nullable
    BeanFactory getParentBeanFactory();

    // 判断指定的 bean 是否注册到容器中, 会忽略祖先容器中定义的 bean, 是 containsBean 的替代方案
    boolean containsLocalBean(String name);
}
```