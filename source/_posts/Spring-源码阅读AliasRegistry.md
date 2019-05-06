---
title: Spring-源码阅读AliasRegistry
date: 2019-04-14 21:16:07
categories: Spring
---

![xmlbeanfactory.png](https://cdn.nlark.com/yuque/0/2019/png/259000/1554948615958-78e34256-49aa-491c-a8f8-de0cc4fb48a1.png#align=left&display=inline&height=350&name=xmlbeanfactory.png&originHeight=652&originWidth=1388&size=42765&status=done&width=746)

<!-- more -->

# AliasRegistry

实现了 AliasRegistry 接口的 BeanFactory 需要实现管理 bean 的别名的功能, 源码如下:

```java
public interface AliasRegistry {
    // 注册一个别名
    void registerAlias(String name, String alias);

    // 移除所有指定的别名
    void removeAlias(String alias);
  
    // 判断 name 是否是一个别名
    boolean isAlias(String name);

    // 返回所有注册的别名
    String[] getAliases(String name);
}
```