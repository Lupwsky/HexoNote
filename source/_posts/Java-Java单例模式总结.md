---
title: Java-Java单例模式总结
date: 2019-05-23 14:31:51
categories: Java
---

# 饿汉模式

* 优点 : 实现简单, 多线程安全问题
* 缺点 : 无法实现延迟加载, 如果单实例非常多, 项目启动时负担大

```java
public class Singleton {
    private static Singleton ourInstance = new Singleton();

    public static Singleton getInstance() {
        return ourInstance;
    }

    private Singleton() {}
}
```

<!-- more -->

# 懒汉模式-单线程

* 优点 : 延迟创建对象, 减轻启动时负担
* 缺点 : 多线程情况下不安全, 两个线程同时获取对象会重复创建对象
* 缺点 : JDK 1.5 前不适用, 由于指令排序优化, 编译器只保证程序执行结果与源代码相同, 却不保证实际指令的顺序与源代码相同, volatile 关键字可以禁止指令排序优化, 但是在 JDK 1.5 之后这一功能才能正确执行
  
```java
public class Singleton {
    private static Singleton ourInstance = null;

    public static Singleton getInstance() {
        if (ourInstance == null) {
            return new Singleton();
        } else {
            return ourInstance;
        }
    }

    private Singleton() {}
}
```

# 懒汉模式-多线程

* 优点 : 延迟创建对象, 减轻启动时负担, 多线程安全
* 缺点 : 存在性能问题, 每次获取对象实例前需要获取锁, 加锁和释放锁, 高并发情况下会存在性能问题
* 缺点 : JDK 1.5 前不适用, 由于指令排序优化, 编译器只保证程序执行结果与源代码相同, 却不保证实际指令的顺序与源代码相同, volatile 关键字可以禁止指令排序优化, 但是在 JDK 1.5 之后这一功能才能正确执行

```java
public class Singleton {
    private static volatile Singleton ourInstance = null;

    public static Singleton getInstance() {
        synchronized (Singleton.class) {
            if (ourInstance == null) {
                return new Singleton();
            } else {
                return ourInstance;
            }
        }
    }

    private Singleton() {}
}
```

# 懒汉模式-多线程安全并提高效率(双重检测)

* 优点 : 延迟创建对象, 减轻启动时负担, 多线程安全, 性能高
* 缺点 : JDK 1.5 前不适用, 由于指令排序优化, 编译器只保证程序执行结果与源代码相同, 却不保证实际指令的顺序与源代码相同, volatile 关键字可以禁止指令排序优化, 但是在 JDK 1.5 之后这一功能才能正确执行
  
```java
public class Singleton {
    private static volatile Singleton ourInstance = null;

    public static Singleton getInstance() {
        synchronized (Singleton.class) {
            if (ourInstance == null) {
                return new Singleton();
            } else {
                return ourInstance;
            }
        }
    }

    private Singleton() {}
}
```

# 内部静态类实现单例模式

* 优点 : 延迟创建对象, 减轻启动时负担, 多线程安全, 性能高, 在 JDK 1.5 之前也可用
* 缺点 : 反序列化时

```java
public class Singleton {
    public static Singleton getInstance() {
        return Holder.ourInstance;
    }

    // 静态内部类只会被加载一次, 可以避免多线程问题
    private static class Holder {
        private static Singleton ourInstance = new Singleton();
    }

    private Singleton(){}
}
```

# 枚举实现单例模式

优点 : 延迟创建对象, 减轻启动时负担, 多线程安全, 性能高, 防止反射调用构造器, 防止反列化时创建新对象
缺点 : 可读性不高, 并且由于用于创建的单例的对象的构造方法不是 private 修饰的, 表明在程序的任意地方都可以去 new 出新的实例出来

```java
public class Singleton {
    // 不是 private 修饰的构造方法
    public Singleton(){}
}

public enum  EnumSingleton {
    // 实例被保证只会被实例化一次
    INSTANCE;

    private Singleton singleton;

    EnumSingleton() {
        singleton = new Singleton();
    }

    public Singleton getInstance() {
        return singleton;
    }
}

// 使用方式
 Singleton singleton = EnumSingleton.INSTANCE.getInstance();
```

实际业务开发中需要更具具体情况选择不同的单例模式实现, 例如在进行多线程, 并需要对单实例对象进行反序列化情况下, 就需要选择枚举的方式来实现

# 参考资料 

* [你真的会写单例模式吗-Java实现](http://www.importnew.com/18872.html#comment-767337) - 多种单例模式的实现
* [Java 利用枚举实现单例模式](https://blog.csdn.net/yy254117440/article/details/52305175) - 枚举类型创建单例模式原理
