---
title: Java-Java8接口的增强
date: 2018-10-10 23:56:31
categories: Java
---

Java 8 对接口做了进一步的增强, 在接口中可以添加 default 关键字修饰方法, 还可以在接口中定义静态方法, 使得接口和抽象类的功能越来越相似。使用 default 修饰的方法也称为扩展方法, 他的使用方法类似于抽象类中的非抽象成员方法, 但是扩展方法不能承载 Object 中的方法, 例如 toString, equals 和 hashCode等。

# default 关键字

在接口中定义一个扩展方法 count, 该方法可以在其子类中直接使用, 示例代码如下:

```java
public interface DefaultFunInterface {
    // 定义默认方法 count
    default int count(){
        return 1;
    }
}
```

<!-- more -->

子类对象可以直接调用父接口中的默认方法 count, 示例代码如下:

```java
public class SubDefaultFunClass implements DefaultFunInterface {
    public static void main(String[] args){
        // 实例化一个子类对象, 子类对象可以直接调用父接口中的默认方法 count
        SubDefaultFunClass sub = new SubDefaultFunClass();
        sub.count();
    }
}
```

# 在接口中使用静态方法

在Java 8中可以使用静态方法, 接口中的静态方法可以直接使用接口来调用, 示例代码如下:

```java
public interface StaticFunInterface {
    public static int find(){
        return 1;
    }
}

public class TestStaticFun {
    public static void main(String[] args){
        // 接口中定义了静态方法 find 直接被调用
        StaticFunInterface.fine();
    }
}
```