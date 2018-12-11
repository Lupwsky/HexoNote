---
title: Java-静态代理和动态代理
date: 2018-12-11 23:03:55
categories:  Java
---

# 代理模式简介

代理模式使设计模式的一种, 为其他对象提供一种代理以控制对这个对象的访问, 这样可以在不修改原目标对象的前提下, 提供额外的功能操作, 扩展目标对象的功能, 代理模式有以下四个角色:

* Subject: 定义代理类和真实主题的公共对外方法, 委托类合代理类都需要实现这个接口
* RealSubject: 真正实现业务逻辑的类, 称之为委托类
* Proxy: 用来封装真实主题的类 (持有委托类的引用, 并加入其他的业务处理), 称之为代理类
* Client: 调用代理类完成业务操作的类

四个角色对应的 UML 关系图如下:

<!-- more -->

## 代理模式优点

* 业务拆分解耦方便
* 具备高扩展性

## 代理模式缺点

* 需要额外的代理类, 随着业务增加, 类的文件会大量增加, 不易维护

# 静态代理

静态代理的代理即代理模式中提到的代理, 主要在与静态, 在 Java 中的这里的静态也就是在程序运行前就已经存在代理类的字节码文件, 即这个代理类是由程序员直接写在 class 文件中的, 在程序运行前委托类和代理类的关系就已经确定, 而动态代理类是在程序运行期间由 JVM 根据反射等机制动态的生成, 所以不存在代理类的字节码文件, 代理类和委托类的关系是在程序运行时确定, 一个简单 Java 静态代理使用示例如下

## 创建接口类

```java
public interface SubjectInterface {
    public void print();
}
```

## 创建委托类

```java
@Slf4j
public class RealSubjectClass implements SubjectInterface {
    @Override
    public void print() {
        log.info("RealSubjectClass.print() 已经调用");
    }
}
```

## 创建代理类

```java
public class ProxyClass implements SubjectInterface {
    private RealSubjectClass realSubjectClass;

    public ProxyClass(RealSubjectClass realSubjectClass) {
        this.realSubjectClass = realSubjectClass;
    }

    @Override
    public void print() {
        log.info("ProxyClass.print() 开始调用");
        realSubjectClass.print();
        log.info("ProxyClass.print() 完成调用");
    }
}
```

## 客户端调用代理类完成相关业务操作

```java
ProxyClass proxyClass = new ProxyClass(new RealSubjectClass());
proxyClass.print();
```

# JDK 原生动态代理

JDK 原生动态代理设计是按照代理模式设计的, 原理是使用反射的方式实现, 代理类需要实现一个接口, 同时委托类是通过 java.lang.reflect 包中的 InvocationHandler 接口和 Proxy 类动态生成的, JDK 原生动态代理使用示例如下

## 创建接口类

```java
public interface SubjectInterface {
    public void print();
}
```

## 创建委托类

实现一个代理类, 完成实际的业务处理, 代码如下:

```java
@Slf4j
public class RealSubjectClass implements SubjectInterface {
    @Override
    public void print() {
        log.info("RealSubjectClass.print() 已经调用");
    }
}
```

## 继承 InvocationHandler 实现生成代理类的程序处理器

```java
@Slf4j
public class ProxyInvocationHandler implements InvocationHandler {

    private RealSubjectClass realSubjectClass;

    public ProxyInvocationHandler(RealSubjectClass realSubjectClass) {
        this.realSubjectClass = realSubjectClass;
    }

    /**
     * 重写 invoke 方法, 加入横切业务逻辑, 如果不对 invoke 接口方法做任何其他处理, 这个横切逻辑会调用对代理类内的所有方法都有效
     *
     * @param proxy 被代理的对象
     * @param method 要调用的方法
     * @param args 方法调用时所需要的参数
     * @return
     * @throws Throwable
     */
    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        log.info("RealSubjectClass.print() 开始调用");
        method.invoke(realSubjectClass, args);
        log.info("RealSubjectClass.print() 结束调用");
        return realSubjectClass;
    }
}
```

## 使用 Proxy 生成代理类

```java
RealSubjectClass realSubjectClass = new RealSubjectClass();
RealSubjectInvocationHandler handler = new RealSubjectInvocationHandler(realSubjectClass);
SubjectClass subjectClass = (SubjectClass) Proxy.newProxyInstance(realSubjectClass.getClass().getClassLoader(),
        realSubjectClass.getClass().getInterfaces(), handler);
subjectClass.print();
```

## 测试输出的日志

```text
[main] INFO com.thread.excutor.proxy.RealSubjectInvocationHandler - RealSubjectClass.print() 开始调用
[main] INFO com.thread.excutor.proxy.RealSubjectClass - RealSubjectClass.print() 已经调用
[main] INFO com.thread.excutor.proxy.RealSubjectInvocationHandler - RealSubjectClass.print() 结束调用
```

## 原生动态代理缺点

代理类和委托类需要都实现同一个接口, 对于没有实现接口的类, 就不能使用动态代理

# CGLIB 动态代理

CGLIB 是一个高性能的代码生成包, JDK 原生的动态代理遵循代理模式的设计, 代理类需要实现接口, 对于没有实现接口的代理类就不能使用 JDK 原生的动态代理方式生成委托类, CGLIB 可以为没有实现接口的类提供代理 (既可以面向类, 也可以面向接口), 对 JDK 原生的动态代理是一个很好的补充, CGLIB 原理是使用继承的方法来实现 (CGLIB 底层使用字节码处理框架 ASM, 来转换字节码并生成新的类), 会动态生成一个代理类的子类, 该子类会重写所有代理类中不是 final 的方法, 然后使用拦截器的拦截方法来织入横切逻辑, 由于产用的是继承, 因此也不能对 final 修饰的类进行处理,同时由于没有使用反射, CGLIB 比 JDK 原生的动态代理性能要好, CGLIB 动态代理使用示例如下

## 创建委托类

```java
@Slf4j
public class RealSubjectClass {
    @Override
    public void print() {
        log.info("RealSubjectClass.print() 已经调用");
    }
}
```

## 创建拦截器

```java
@Slf4j
public class ProxyInterceptor implements MethodInterceptor {

    /**
     * 重写拦截方法, 在代理类方法中加入业务
     *
     * @param o           目标对象
     * @param method      目标方法
     * @param objects     为参数
     * @param methodProxy CGlib方法代理对象
     * @return
     * @throws Throwable
     */
    @Override
    public Object intercept(Object o, Method method, Object[] objects, MethodProxy methodProxy) throws Throwable {
        log.info("RealSubjectClass.print() 开始调用");
        // 调用代理类实例上的父类方法
        Object result = methodProxy.invokeSuper(o, objects);
        log.info("RealSubjectClass.print() 结束调用");
        return result;
    }
}
```

## 生成代理类并调用

```java
// enhancer 中文翻译为增强, 扩展
Enhancer enhancer =new Enhancer();
enhancer.setSuperclass(RealSubjectClass.class);
enhancer.setCallback(new ProxyInterceptor());
RealSubjectClass realSubjectClass =(RealSubjectClass)enhancer.create();
RealSubjectClass.print();
```

# 静态代理优缺点

* 解耦, 如实现业务层和数据层的分离
* 扩展性高
* 随着业务的复杂度增加, 会产生过多的代理类
* 不易维护, 接口增加方法, 委托类和代理类都需要更改, 随着业务复杂度增加该问题更加凸显

# 动态代理优缺点

* 解耦, 如实现业务层和数据层的分离
* 扩展性高
* 可以实现无侵入式的代码扩展, 随着业务复杂程度增加, 代理类不会因此大量增加
* 可以实现 AOP 编程, 静态代理方式难以实现, Spring AOP 就使用了动态代理
* 使用了反射等生成代理类, 性能相比静态代理性能要低
