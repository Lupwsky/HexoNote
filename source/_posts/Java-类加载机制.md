---
title: Java-类加载机制
date: 2018-10-09 23:04:01
categories:  Java
---

# 读取resource目录下的配置文件

说一说了解 ClassLoader 的缘由, 在准备开始看 Spring 源码的时候想把 Properties 类的使用了解下, 于是我建立了一个普通的 Java 项目写下如下的这些代码:

```java
public class Main {
    public static void main(String[] args) {
    Properties properties = new Properties();
    try {
        InputStream fileInputStream = new FileInputStream("D:\\IdeaProjects\\hello-grpc\\grpc-client\\src\\main\\resources\\application.properties");
        properties.load(fileInputStream);
    } catch (IOException e) {
        e.printStackTrace();
    }
    System.out.println(properties.getProperty("spring.application.name"));
}
```

可以正确的读取和获取到 peoperties 文件中的值, 这里使用了 FileInputStream 类来获取输入流， 需要指定配置文件的路径, 这种方法不推荐使用, 因为很不方便, 通常我们在项目中读取配置文件中的内容, 在本地调试的时候需要使用本地的路径, 正式上传到服务器的时候又需要单独的上传这些配置文件和更改成服务器上配置文件的路径, 每次在本地调试和上传到服务器发布很不方便, 需要检查所有读取配置文件的代码, 很容易漏掉, 况且 JavaWeb 项目的资源文件和配置文件通常放在项目的 resource 目录下, 在编译后打包成 war 和 jar 时这些资源文件也会打包进去, 完全没有理由再去服务器上创建一个专门的目录保存这些资源文件, 在了解相关资料后可以使用 ClassLoader 的 getResourceAsStream() 方法来获取输入流, 在未了解类加载机制的情况下我写下了如下代码:

```java
Properties properties = new Properties();
InputStream inputStream = Application.class.getClassLoader().getResourceAsStream("D:\\IdeaProjects\\hello-grpc\\grpc-client\\src\\main\\resources\\application.properties");
try {
    properties.load(inputStream);
} catch (IOException e) {
    e.printStackTrace();
}
log.info(properties.getProperty("spring.application.name"));
```

运行后, 这段代码报错了, 空指针异常, 没有读到配置文件的内容, 在网上查阅资料后, 将代码更改如下, 顺利的读取到配置文件中的值:

```java
InputStream inputStream = Application.class.getClassLoader().getResourceAsStream("application.properties");
```

为什么获取到 ClassLoader 后不需要指定文件的路径就可以通过 getResourceAsStream() 方法顺利的读取到配置文件呢? 要理解这个问题需要了解虚拟机类加载机制的相关知识, 这里先简单了解下类的加载过程, 更加详细的内容可参考 `深入了解Java虚拟机-虚拟机的加载机制` 一章

# 简单了解类加载机制

虚拟机将描述类的数据从 class 文件加载到内存, 并对数据进行校验, 解析和初始化直到形成可被虚拟机直接使用的 Java 类型的过程称之为虚拟机的类的加载机制

## 类的加载主要过程

从类被加载虚拟机的内存中开始到它从内存中卸载, 它的生命周期主要包括: `加载` -> `连接` -> `初始化` -> `使用` -> `卸载` 五个过程, 其中 `验证` 又分成 `准备` -> `解析` 三个过程

## 类加载器

使用类加载器可以将类加载到虚拟机的内存中, Java 自带有三个类加载器, BootstrapClassLoader, ExtentionClassLoader 和 AppClassLoader, 这三个类的功能如下:

* BootstrapClassLoader: 在虚拟机启动的时候自动加载 `%JRE_HOME%/lib` 目录下的 jar 包和 class 类, 并且只加载里面特定文件名的 jar 包和 class 类, 由 c++ 实现, 是虚拟机的一部分
  
* ExtentionClassLoader: 在虚拟机启动的时候自动加载 `%JRE_HOME%/lib/ext` 目录下额外的 jar 包和 class 类

* AppClassLoader: 在虚拟机启动的时候自动加载当前应用的 classpath 目录下的所有类, 即编译后存放 class 文件 (资源文件也回保存在这) 的路径, 如 JavaWeb 是在 target/class 目录下

其启动加载的过程可参考 `sun.misc.Launcher` 类