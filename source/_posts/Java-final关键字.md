---
title: Java-final关键字
date: 2018-05-17 11:34:00
categories: Java
---

final 可以声明成员变量、方法、类以及本地变量，如果修饰的成员变量非基本类型(int，long等)，你将不能改变这个引用了，这个时候编译器也会报编译错误。在高并发的场景中使用 fianl 可以提高性能，并减少额外的开销。

# final 变量

对成员变量或者本地变量声明为 final 的都叫作 final 变量，final变量经常和static关键字一起使用，作为常量。final 变量只是可读的，如果是引用，引用的内容可变，但是是引用不可变。

# final 方法

final 也可以声明方法，代表这个方法不可以被子类的方法重写。

# final 类

使用 final 来修饰的类叫作 final 类，它们不能被继承，Java 中有许多类是 final 的，如 String, Interger等。

<!-- more -->

<div class="note default"><p> 有个类既然被声明成 final，表示这个类的功能是完整的，不需要继承和扩展了，因此 final 和 abstract 这两个关键字是完全相反的，final 类就不可能是 abstract 的类 </p></div>

# final 参数

在实际应用中，我们除了可以用 final 修饰成员变量、成员方法、类，还可以修饰参数、若某个参数被final修饰了，则代表了该参数是不可改变的。**final 修饰参数在内部类中是非常有用的，在匿名内部类中，为了保持参数的一致性，若所在的方法的形参需要被内部类里面使用时，该形参必须为 final**

# final、finally 和 finalize 方法

## finally

finally 是保证重点代码一定要被执行的一种机制，如用于回收资源，常用于 try-finally 或者 try-catch-fianlly 语句中。

### finally 一定会执行么

try-catch-finally 块中的 finally 语句是不是一定会被执行，答案是否定的，至少以下两个个例子是不运行的。

在执行 try 语句之前就返回了的伪代码：

```java
public void runMethod() {
    if(...) {
        return;
    }

    try {
        // do something
    } catch (Exception e) {
        // do something
    } finally {
        // do something
    }
}
```

在执行 try 语句时，JVM 停止运行的伪代码：

```java
public void runMethod() {
    try {
        System.exit(0);
    } catch (Exception e) {
        // do something
    } finally {
        // do something
    }
}
```

<div class="note default"><p> 线程在执行 try-catch-finally 语句的时候线程被杀死也不会执行 fainally 语句 </p></div>

### final 相关的资料

Java finally语句到底是在return之前还是之后执行：`http://www.cnblogs.com/lanxuezaipiao/p/3440471.html`

## finalize

finallize 是 Object 类里面的一个方法，该方法在 Object 类里面默认没有任何实现，该方法的作用是实例被垃圾回收器回收的时候触发的操作，以便回收资源，但是并不推荐使用，因为垃圾收集的时间是不确定的，因为对于资源的使用，大部分都是有限的，用完之后都是希望及时回收这些有限的资源。从 JDK 9 开始，这个方法也是开始被标记为废弃的方法。