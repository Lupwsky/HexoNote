---
title: Java-反射
date: 2018-06-25 22:11:54
categories: Java
---

在看 Spring 框架的源码时，需要 Java 反射相关的知识，了解反射知识时网上看到几篇反射相关的文章，个人感觉挺不错的，记录如下：

* [JAVA中的反射 超级详解](https://blog.csdn.net/qq_24341197/article/details/77964172)
* [深入理解 Java 反射：Field (成员变量)](https://blog.csdn.net/u011240877/article/details/54604212)
* [深入理解 Java 反射：Method (成员方法)](https://blog.csdn.net/u011240877/article/details/54604224)

<!-- more -->

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