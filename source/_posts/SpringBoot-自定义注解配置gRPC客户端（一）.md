---
title: SpringBoot-自定义注解配置gRPC客户端（一）
date: 2018-09-05 23:04:35
categories:  Spring
---

# 基于注解自动配置的 gRPC 客户端实现（一）

基于注解自动配置的gRPC 的客户端的实现相较于服务端实现起来复杂一些，客户端除了可以绑定拦截器，还可以根据需要实现 NameResoler 和 LoadBalancer 等，先从简单的来慢慢实现 (主要用于捋清思路)。先看一看一个最简单的 gRCP 客户端调用 gRPC 服务端的例子，这个简单例子完整的使用在[gRPC的使用](http://antsnote.club/2018/06/21/SpringBoot-gRPC%E7%9A%84%E4%BD%BF%E7%94%A8/)笔记中有记录。

```java
public class MainClient {
    public static void main(String[] args) {
        // 创建 channel
        ManagedChannel channel = ManagedChannelBuilder.forAddress("127.0.0.1", 8000)
                .usePlaintext(true)
                .keepAliveWithoutCalls(true)
                .keepAliveTimeout(120, TimeUnit.SECONDS)
                .build();

        // 使用 channel 创建 BlockingStub 后就可以调用服务提供的方法
        TestServiceGrpc.TestServiceBlockingStub blockingStub = TestServiceGrpc.newBlockingStub(channel);
        Response response = blockingStub.test(Request.newBuilder().setId(1).build());
        System.out.println("id = " + response.getId());
        System.out.println("name = " + response.getName());
    }
}
```

<!-- more -->


# 定义配置类

客户端调用服务端，首先需要获取到 Channel 实例，最简单的，至少要连接的服务器的 IP 和端口号这两个参数，所以我们先定义一个配置类，通过 ConfiggurationPropertoes 配置文件的中的属性映射成 POJO 类。

```java
@Data
@Configuration
@ConfigurationProperties("grpc.client")
public class GrpcChannelProperties {
    private String host;
    private int port;
}
```

这样就可以在 application.properties 配置文件中配置文件的属性值。

```ini
grpc.client.host=127.0.0.1
grpc.client.port=6000
```

# 定义注解类

接着创建一个注解，将 Channel 注入到需要依赖 Channel 的类中。

```java
@Target({ElementType.FIELD})
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
public @interface GrpcClient {
}
```

# 核心代码实现类

接下来就是关键的实现代码了，之前为了让项目在启动的时候自动创建 gRPC 的服务，使用了 CommandLineRunner 类来实现，为了保证客户端启动的时候服务端是已经启动的，gRCP 这里的实现就不能使用这种方式，这里采用继承 BeanPostProcessor 类来实现，具体的实现如下：

```java
@EnableConfigurationProperties(value = {GrpcChannelProperties.class})
public class GrpcClientBeanPostProcessor implements org.springframework.beans.factory.config.BeanPostProcessor {

    private Map<String, List<Class>> beansToProcess = new HashMap<>();

    @Autowired
    private GrpcChannelProperties channelProperties;

    // 在 bean 创建之前调用
    @Override
    public Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
        Class clazz = bean.getClass();
        do {
            // 遍历这个 Bean 类的所有字段，如果这个 Bean 里面有字段使用了注解 @GrpcClient
            // 添加到以 beanName 为 key 值的 Map 中
            for (Field field : clazz.getDeclaredFields()) {
                if (field.isAnnotationPresent(GrpcClient.class)) {
                    if (!beansToProcess.containsKey(beanName)) {
                        beansToProcess.put(beanName, new ArrayList<Class>());
                    }
                    beansToProcess.get(beanName).add(clazz);
                }
            }
            clazz = clazz.getSuperclass();
        } while (clazz != null);
        return bean;
    }

    // 在 bean 创建之后调用
    @Override
    public Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {
        if (beansToProcess.containsKey(beanName)) {
            Object target = getTargetBean(bean);
            for (Class clazz : beansToProcess.get(beanName)) {
                // 循环遍历类的字段，找到使用了 @GrpcClient 注解的字段
                for (Field field : clazz.getDeclaredFields()) {
                    GrpcClient annotation = AnnotationUtils.getAnnotation(field, GrpcClient.class);
                    if (null != annotation) {
                        // 创建 channel，channel.value() 是客户端需要连接的服务器端名称
                         Channel channel = ManagedChannelBuilder.forAddress(channelProperties.getHost(), channelProperties.getPort())
                                 .usePlaintext(true)
                                 .keepAliveWithoutCalls(true)
                                 .keepAliveTimeout(120, TimeUnit.SECONDS)
                                 .build();
                        // 暴力访问 field 字段
                         ReflectionUtils.makeAccessible(field);
                        // 设置 field 的值，将 channel 的值设置和这个字段
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

# 使用方法

使用方法就很简单了，实例代码如下：

```java
@Slf4j
@Service
public class TestServiceGrpcImpl {

    @GrpcClient
    private Channel channel;

    public void getUserInfo() {
        UserServiceGrpc.UserServiceBlockingStub stub = UserServiceGrpc.newBlockingStub(channel);
        Response resp = stub.getUserInfo(Request.newBuilder().setId(1).build());
        log.info("name = {}, id = {]", resp.getName(), resp.getId());
    }
}
```