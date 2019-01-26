---
title: Java-反射
date: 2018-06-25 22:11:54
categories: Java
---

反射是 Java 的特征之一, 它允许运行中的 Java 程序获取自身的信息, 并且可以操作类或对象的内部属性, 通过反射，我们可以在运行时获得程序或程序集中每一个类型的成员和成员的信息, 程序中一般的对象的类型都是在编译期就确定下来的, 而 Java 反射机制可以动态地创建对象并调用其属性, 这样的对象的类型即使在编译期是未知的, 可以通过反射机制直接创建对象

# 反射的作用

* 在运行时判断任意一个对象所属的类
* 在运行时创建任意一个类的对象
* 在运行时调设置任意一个类所具有的成员变量
* 在运行时调调用任意一个类所具有的和方法

<!-- more -->

# 反射的使用

反射使用一般的步骤, 第一步需要先获取 Class 对象实例, 获取对象实例就可以进行创建类的实例, 设置已经实例化对象成员变量, 或者调用方法等操作, 在进行之后一系列方法测试时, 先建立如下四个测试类

```java
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Target(ElementType.FIELD)
public @interface UserAnnotation {
    String value() default "";
}

@Data
public class UserInfo {
    private String name;
    private String age;
}

public class SuperReflectionClazz {
    private UserInfo superUserInfo;

    private String superPrivateMember;

    private final String superPrivateFinalMember = "superPrivateFinalMemberValue";

    public String superPublicMember;

    public final String superPublicFinalMember = "superPublicFinalMemberValue";

    public String superFunc1(String num, String value) {
        return num + value;
    }

    private String superFunc2(String num, String value) {
        return num + value;
    }
}

public class ReflectionClazz extends SuperReflectionClazz {
    public static final String CLASS_NAME = "com.lupw.guava.reflection.ReflectionClazz";

    @UserAnnotation(value = "userAnnotationTest")
    private static UserInfo userInfo;

    private String privateMember;

    private final String privateFinalMember = "privateFinalMemberValue";

    public String publicMember;

    public final String publicFinalMember = "publicFinalMemberValue";

    public String func1(String num, String value) {
        return num + value;
    }

    private String func2(String num, String value) {
        return num + value;
    }
}
```

## 获取 Class 对象实例

```java
// (1) Class.forName("全类名"), 代理类可以在获取到全类名后可以使用这种方法
Class clazz = Class.forName(ReflectionClazz.CLASS_NAME);

// (2) 知道具体类时, 类.class 和 Class 实例的 getComponentType() 方法均可获取到
Class clazz1 = ReflectionClazz.class;

// (3) 有的类有一个静态属性 TYPE, 也可以获取到 Class 对象实例
Class clazz2 = Integer.TYPE;
```

## 实例化对象

```java
// (1) Class 对象的 newInstance() 方法来创建
Class<?> reflectionClazz = ReflectionClazz.class;
ReflectionClazz str = reflectionClazz.newInstance();

// 获取 Constructor 对象后使用 Constructor 创建
Class<?> reflectionClazz = ReflectionClazz.class;
Constructor constructor = reflectionClazz.getConstructor(String.class, String.class);
ReflectionClazz reflectionObject = (ReflectionClazz) constructor.newInstance("lpw", "soga");
```

## 获取成员变量

getDeclaredField(String fieldName) 和 getField(String fieldName) 方法获取到成员变量的实例对象, Field 字段包含了成员变量的各种属性

```java
// 获取显示声明指定字段, private, public 的都会返回
Field field = clazz.getDeclaredField("userInfo");
// 成员名称
log.info("[REFLECTION] name = {}", field.getName());

// 成员类型
log.info("[REFLECTION] type = {}", field.getType().getName());
log.info("[REFLECTION] genericType = {}", field.getGenericType().getTypeName());
log.info("[REFLECTION] annotatedType = {}", field.getAnnotatedType().getType());

// 获取成员添加注解, 如果没有就返回 null, getAnnotations() 可以获取全部添加在上面的注解
log.info("[REFLECTION] annotationValue = {}", field.getAnnotation(UserAnnotation.class).value());

// 修饰符
log.info("[REFLECTION] modifiersIndex = {}", field.getModifiers());
log.info("[REFLECTION] modifiersValue = {}", Modifier.toString(field.getModifiers()));
log.info("[REFLECTION] modifiersIsPrivate = {}", Modifier.isPrivate(field.getModifiers()));
log.info("[REFLECTION] modifiersIsStatic = {}", Modifier.isPrivate(field.getModifiers()));

// 返回当前类 (不包括父类) 声明所有字段, private 和 public 都会返回
Field[] declaredFields = clazz.getDeclaredFields();
Arrays.stream(declaredFields).forEach(declaredField -> {
    log.info("[REFLECTION] name = {}", declaredField.getName());
});

// 返回当前类和父类使用 public 声明的所有字段
Field[] fields = clazz.getFields();
Arrays.stream(fields).forEach(tempField -> {
    log.info("[REFLECTION] name = {}", tempField.getName());
});
```

输出结果如下:

```text
com.lupw.guava.reflection.ReflectionMain - [REFLECTION] modifiersIsStatic = true
com.lupw.guava.reflection.ReflectionMain - [REFLECTION] name = CLASS_NAME
com.lupw.guava.reflection.ReflectionMain - [REFLECTION] name = userInfo
com.lupw.guava.reflection.ReflectionMain - [REFLECTION] name = privateMember
com.lupw.guava.reflection.ReflectionMain - [REFLECTION] name = privateFinalMember
com.lupw.guava.reflection.ReflectionMain - [REFLECTION] name = publicMember
com.lupw.guava.reflection.ReflectionMain - [REFLECTION] name = publicFinalMember
com.lupw.guava.reflection.ReflectionMain - [REFLECTION] name = CLASS_NAME
com.lupw.guava.reflection.ReflectionMain - [REFLECTION] name = publicMember
com.lupw.guava.reflection.ReflectionMain - [REFLECTION] name = publicFinalMember
com.lupw.guava.reflection.ReflectionMain - [REFLECTION] name = superPublicMember
com.lupw.guava.reflection.ReflectionMain - [REFLECTION] name = superPublicFinalMember
```

## 获取方法

和获取成员变量, getMethod() 可以获取到 Method 实例, 包含了方法信息, 大多方法调用和 Field 一样, 不再去举例了, 主要有一个要注意的地方是, 获取成员方法的参数名, 如下实例所示:

```java
// 获取方法和参数名
Method method = clazz.getMethod("func1", String.class, String.class);
Parameter[] parameter = method.getParameters();
log.info("[REFLECTION] parameter0 = {}, parameter1 = {}", parameter[0].getName(), parameter[1].getName());
```

输出结果如下:

```text
com.lupw.guava.reflection.ReflectionMain - [REFLECTION] parameter0 = num, parameter1 = value
```

# 实际使用

在 Spring 项目中使用 gRPC 时，为了方便使用，自定义了注解 @GrpcClient，为了在应用启动的时候，找到使用了注解 @GrpcClient 的 Bean，进行处理，使用到了反射的知识。

```java
public class GrpcClientBeanPostProcessor implements BeanPostProcessor {

    private Map<String, List<Class>> beansToProcess = new HashMap<>();

    @Autowired
    private DefaultListableBeanFactory beanFactory;

    @Autowired
    private GrpcChannelFactory channelFactory;

    public GrpcClientBeanPostProcessor() {
    }

    @Override
    public Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
        // 获取字节码
        Class clazz = bean.getClass();
        do {
            // 循环遍历 Bean 的成员属性，使用了 @GrpcClient 注解的 Bean，添加到 Map 中
            for (Field field : clazz.getDeclaredFields()) {
                if (field.isAnnotationPresent(GrpcClient.class)) {
                    if (!beansToProcess.containsKey(beanName)) {
                        beansToProcess.put(beanName, new ArrayList<Class>());
                    }
                    beansToProcess.get(beanName).add(clazz);
                }
            }
            // 遍历完成后继续遍历父类的成员属性
            clazz = clazz.getSuperclass();
        } while (clazz != null);
        return bean;
    }

    @Override
    public Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {
        if (beansToProcess.containsKey(beanName)) {
            Object target = getTargetBean(bean);
            for (Class clazz : beansToProcess.get(beanName)) {
                for (Field field : clazz.getDeclaredFields()) {
                    // 获取注解对象
                    GrpcClient annotation = AnnotationUtils.getAnnotation(field, GrpcClient.class);
                    if (null != annotation) {
                        List<ClientInterceptor> list = Lists.newArrayList();
                        // 循环遍历拦截器
                        for (Class<? extends ClientInterceptor> clientInterceptorClass : annotation.interceptors()) {
                            ClientInterceptor clientInterceptor;
                            if (beanFactory.getBeanNamesForType(ClientInterceptor.class).length > 0) {
                                clientInterceptor = beanFactory.getBean(clientInterceptorClass);
                            } else {
                                try {
                                    // 通过反射方式创建拦截器的实例
                                    clientInterceptor = clientInterceptorClass.newInstance();
                                } catch (InstantiationException | IllegalAccessException e) {
                                    throw new BeanCreationException("Failed to create interceptor instance", e);
                                }
                            }
                            list.add(clientInterceptor);
                        }
                        // 创建 Channel，创建 Channel 的代码这里就不贴出了，了解 gRPC 的使用方式就知道了
                        Channel channel = channelFactory.createChannel(annotation.value(), list);
                        ReflectionUtils.makeAccessible(field);
                        ReflectionUtils.setField(field, target, channel);
                    }
                }
            }
        }
        return bean;
    }

    @SneakyThrows
    private Object getTargetBean(Object bean) {
        Object target = bean;
        while (AopUtils.isAopProxy(target)) {
            target = ((Advised) target).getTargetSource().getTarget();
        }
        return target;
    }
}
```

定义的 @GrpcClient 注解代码如下：

```java
@Target({ElementType.FIELD})
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
public @interface GrpcClient {
    String value();
    Class<? extends ClientInterceptor>[] interceptors() default {};
}
```

# 参考资料

* [JAVA中的反射 超级详解](https://blog.csdn.net/qq_24341197/article/details/77964172)
* [Java基础-反射](https://blog.csdn.net/sinat_38259539/article/details/71799078)
* [深入理解 Java 反射：Field (成员变量)](https://blog.csdn.net/u011240877/article/details/54604212)
* [深入理解 Java 反射：Method (成员方法)](https://blog.csdn.net/u011240877/article/details/54604224)