---
title: Java-注解和注解在SpringAOP中的使用示例
date: 2019-01-26 21:01:55
categories: Java
---

注解和 class, interface 一样属于一种类型, 在 Spring 中, 注解就被经常使用, 最典型的用法就是注解来注入属性值, 日常开发中我也经常会用到反射和注解配合使用的的方式动态的处理一些代码, 来完成某些业务代码的解耦

# 注解的作用

* 编写文档, 如 JDK 中用于帮助生成文档的注解 @param, @return
* 编译检测, 让编译器能实现基本的编译检查, 如 @Override 注解在重写父类父类方法时帮助检查方法的正确性, 如果不是重写的父类的方法, 则编译器会报错
* 动态处理代码, 如 Spring 使用反射 + 注解配合使用完成依赖注入

<!-- more -->

# 注解的定义

class 定义一个类, interface 定义一个接口, @interface 定义一个注解, 示例如下:

```java
public @interface AopAnnotation {
}
```

# 注解的分类

* 元注解 (注解的注解), @Retention, @Target, @Inherited, @Documented, @Repeatable 五种
* 标准注解, JDK 中使用的注解, 如典型的 @Override, @Deprecated, @SuppressWarnings, @FunctionalInterface (JDK 1.8 新增, 标识接口为函数式接口) 等
* 自定义注解

# 注解的注解

注解的注解, 即元注解也是一种特殊的注解, 用于描述一个普通注解的基础信息, 可以理解为注解的注解, JDK 中一共提供了五种元注解

## @Retention

@Retention 注解描述了一个注解的存活时间, 有以下值, 只能选一个

* RetentionPolicy.SOURCE 注解只在源码阶段保留, 在编译器进行编译时它将被丢弃忽视
* RetentionPolicy.CLASS 注解只被保留到编译进行的时候, 并不会被加载到 JVM 中
* RetentionPolicy.RUNTIME 注解可以保留到程序运行的时候, 会被加载进入到 JVM 中, 在程序运行时可以获取到它们

## @Target

@Target 注解描述了这个注解可以被用到的地方, 如注解可以被用于包上, 类上或者成员属性或成员方法上, 有以下值, 可以配置多个值

* ElementType.ANNOTATION_TYPE 可以给一个注解进行注解
* ElementType.CONSTRUCTOR 可以给构造方法进行注解
* ElementType.FIELD 可以给属性进行注解
* ElementType.LOCAL_VARIABLE 可以给局部变量进行注解
* ElementType.METHOD 可以给方法进行注解
* ElementType.PACKAGE 可以给一个包进行注解
* ElementType.PARAMETER 可以给一个方法内的参数进行注解
* ElementType.TYPE 可以给一个类型进行注解, 比如类, 接口, 枚举

## @Inherited

注解的继承, 一个类注解了使用了@Inherited 注解的注解, 如果这个类被继承, 如果子类没有使用其他任何注解, 就会继承父类的注解

## @Documented

用于文档的生成, 能够将注解中的元素包含到 Javadoc 中去

## @Repeatable

JDK 1.8 新增的注解, 允许我们使用重复的注解, 使用多个值, 例如定义如下一个注解

```java
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@interface AopAnnotation {
    String value() default "";
}
```

如果我们像如下使用, 编译器则会直接报错

```java
@AopAnnotation(value = "A")
@AopAnnotation(value = "B")
public void test(String name, String userId) {
    log.info("[aopTest] name = {}, userId = {}", name, userId);
}
```

创建一个新的 AopAnnotations 注解, 到 AopAnnotation 注解上添加 @Repeatable(AopAnnotations.class) 注解后就可以重复的添加注解了

```java
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface AopAnnotations {
    AopAnnotation[] value();
}
```

# 注解的属性

注解的属性, 使用示例如下, 这样就定义了一个 value 属性, 在使用的时候可以这样为属性赋值 @AopAnnotation(value = "A"), 接着如果我们获取到了注解实例, 那我们就可以通过 aopAnnotation.value() 的方式获取到值 "A"

```java
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface AopAnnotation {
    String value() default "";
}
```

# 在Spring AOP 中使用示例

引入依赖:

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-aop</artifactId>
</dependency>
```

在类上面添加 @Aspect 注解:

```java
@Aspect
@Slf4j
@Component
public class LogAop {

    /**
     * 定义切点
     */
    @Pointcut(value = "@annotation(com.lupw.guava.aspect.AopAnnotation)")
    public void aopAnnotation(){
    }

    @Before(value = "aopAnnotation()")
    public void before(JoinPoint point) {
        // 使用 ProceedingJoinPoint 时 IDEA 提示只能在 @Around 注解的方法中被接收
        // 简单参数值
        Object[] args = point.getArgs();
        Arrays.stream(args).forEach(arg -> log.info("[LogAop] arg = {}", arg));

        // 代理类全类名, 获取到全类名, 就能获取 Class 的实例对象, 也就能使用反射了
        String className = point.getThis().getClass().getName();
        log.info("[LogAop] classType = {}", className);

        // 当前被调用方法的方法信息, 里面有参数名, 参数类型
        MethodSignature methodSignature = (MethodSignature) point.getSignature();
        String[] params = methodSignature.getParameterNames();
        log.info("[LogAop] methodName = {}", methodSignature.getMethod().getName());
        log.info("[LogAop] param0 = {}, param1 = {}", params[0], params[1]);
    }

    /**
     * 在 test 后运行
     * try {
     *     try{
     *         // @Before
     *         method.invoke(..);
     *     }finally{
     *         // @After
     *     }
     *     // @AfterReturning
     * } catch(){
     *     // @AfterThrowing
     * }
     */
    @After(value = "aopAnnotation()")
    public void after(JoinPoint point) {
        log.info("[LogAop] 1----------");
    }

    /**
     * 获取运行之后的返回值
     */
    @AfterReturning(returning = "o", pointcut = "aopAnnotation()")
    public Object afterReturning(JoinPoint point, Object o) {
        log.info("[LogAop] 2----------");
        return null;
    }

    @Around(value = "aopAnnotation()")
    public Object around(ProceedingJoinPoint point) {
        // 在 Before 之前, 并在 After 之后执行
        // @Around("execution(* com.lupw.guava.controller.*(..))")
        Object[] args = point.getArgs();
        try {
            // 获取返回方法返回值, 这里可以对 args 进行修改后再用新的值调用方法
            Object returnValue = point.proceed(args);
        } catch (Throwable throwable) {
            throwable.printStackTrace();
        }

        // 对返回值进行修改后再返回新的值
        return null;
    }
}
```

定义的注解:

```java
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Repeatable(AopAnnotations.class)
@interface AopAnnotation {
    String value() default "";

}

@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface AopAnnotations {
    AopAnnotation[] value();
}
```

在方法上使用添加 @AopAnnotations 注解, 就会调用定义的切点方法:

```java
@Slf4j
@RestController
public class AopTestController {
    @PostMapping(value = "/aop/test")
    @AopAnnotation(value = "A")
    @AopAnnotation(value = "B")
    public void test(String name, String userId) {
        log.info("[aopTest] name = {}, userId = {}", name, userId);
    }
}
```