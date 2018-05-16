---
title: Java-static关键字
date: 2018-05-15 11:45:41
categories: Java
---

Java 中 static 表示【静态】的意思，Java 中没有全局变量，但是可以通过 static 来实现。用来修饰成员变量和成员方法，也可以修饰代码块，分别称之为静态变量、静态方法和静态代码块，被 static 修饰的成员变量是独立于该类的，它被该类的所有实例共享，任何一个实例对其的修改都会导致其他实例的变化。

# static 变量

static修饰的变量我们称之为静态变量，静态变量是随着**类加载时被完成初始化的**，它在内存中仅有一个，且 JVM 也**只会为它分配一次内存**，同时类所有的实例都共享静态变量，可以直接通过类名来访问它。

```java
ClassName.propertyName;
```

# static 方法

static修饰的方法我们称之为静态方法，在**类加载的时候就存在了，static 方法必须实现，也就是说他不能是抽象方法 abstract**，通过类名对其进行直接调用。

```java
ClassName.methodName(args...);
```

<!-- more -->

# static 代码块

被static修饰的代码块，我们称之为静态代码块，静态代码块会**随着类的加载一块执行**，而且他可以随意放在类的任何位置。看一看 Object 的源码中的 static 代码块的使用。

```java
public class Object {
    private static native void registerNatives();
    static {
        registerNatives();
    }
    // ...
}
```

registerNatives() 是一个原生的方法，通过 JNI 的方式调用 Java_java_lang_Object_registerNatives 方法对几个本地方法进行注册，在 Java 中调用这些方法的地方会去调用 c 语言的中的实现方法。

```c
static JNINativeMethod methods[] = {
    {"hashCode",    "()I",                    (void *)&JVM_IHashCode},
    {"wait",        "(J)V",                   (void *)&JVM_MonitorWait},
    {"notify",      "()V",                    (void *)&JVM_MonitorNotify},
    {"notifyAll",   "()V",                    (void *)&JVM_MonitorNotifyAll},
    {"clone",       "()Ljava/lang/Object;",   (void *)&JVM_Clone},
};

Java_java_lang_Object_registerNatives(JNIEnv *env, jclass cls){
    (*env)->RegisterNatives(env, cls, methods, sizeof(methods) / sizeof(methods[0]));
}
```