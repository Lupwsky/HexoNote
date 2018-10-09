---
title: SpringBoot-自定义注解配置gRPC服务端
date: 2018-09-05 23:00:27
categories:  Spring
---

# 基于注解自动配置的 gRPC 服务端

在实际的 SpringBoot 或者项目中, 通常是基于注解和配置的方式实现在项目启动的时候创建 gRPC 服务端, 这里先来实现一个简单的版本。在[SpringBoot-gRPC的使用](http://antsnote.club/2018/06/21/SpringBoot-gRPC%E7%9A%84%E4%BD%BF%E7%94%A8/)笔记中创建一个 gRPC 服务端需要提供端口号, 绑定服务实现类和拦截器实现类，虽然是要实现基于注解和配置的方式实现在项目启动的时候创建 gRPC 服务端功能，但是创建 gRPC 服务端的步骤是一样的，按照这个步骤来一步一步实现即可。

## 定义配置类

为了可以从配置文件中读取到端口号, 创建一个 Porperties 类, 从配置文件中注入属性值创建 bean。

```java
// 设置配置文件,  value 接收 String[] 类型的值, 多个路径 value = {"classpath:xxx", "classpath:xxx"} 的格式设置
@PropertySource(value = "classpath:application.properties")
// 属性值匹配的前缀
@ConfigurationProperties(prefix="grpc.server")
@Data
public class GrpcServerProperties {
    private int port;
}
```

<!-- more -->

## 定义注解类

为了可以使用注解的方式配置 gRPC 服务端的实现类和拦截器, 定义两个注解 @GrpcIntercepter 和 @GrpcService, @GrpcIntercepter 注解表示这是一个会和某个服务实现类绑定的拦截器, 使用这个注解的类需要继承 gRPC 提供的拦截器类。@GrpcService 则表示这个类是一个服务实现类。

```java
@Target({ElementType.TYPE,ElementType.METHOD })
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Component
public @interface GRpcGlobalInterceptor {
}

@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Service
public @interface GrpcService {
    Class<? extends ServerInterceptor>[] interceptors() default {};
    boolean applyGlobalInterceptors() default true;
}
```

## 核心代码实现类

为了弄够在项目启动时就自动创建和初始化 gRPC 的服务端, 需要使用 Spring 的 CommandLineRunner, 创建一个类实现 CommandLineRunner 方法, 并将这个类添加注解 @Configuration, 将这个类作为配置类使用。先来看看一个简单的版本，不绑定拦截器，具体的代码如下，里面的注释很清楚。

```java
@Slf4j
@Configuration
@EnableConfigurationProperties(value = {GrpcServerProperties.class})
public class GrpcServerRunner implements CommandLineRunner, DisposableBean  {

    private final AbstractApplicationContext applicationContext;
    private final GrpcServerProperties grpcServerProperties;
    private Server server;

    @Autowired
    public GrpcServerRunner(AbstractApplicationContext applicationContext, GrpcServerProperties grpcServerProperties) {
        this.applicationContext = applicationContext;
        this.grpcServerProperties = grpcServerProperties;
    }

    @Override
    public void run(String... args) throws Exception {
        log.info("开始创建 gRPC 服务 ...");

        // 扫描出所有的服务实现类
        // 获取所有的类型为 BindableService 类的 bean 的名称，gRPC 的实现类均是 BindableService 的子类
       String[] beanNames = applicationContext.getBeanNamesForType(BindableService.class);
       for (String name : beanNames) {
           log.info(name);
       }

       // 获取所有使用了注解 @GrpcServer 的 bean 类
       Map<String, Object> beansWithAnnotation = applicationContext.getBeansWithAnnotation(GrpcService.class);

       // 类型为 BindableService 且使用了 @GrpcServer 注解的 bean 类，才是一个有效的 gRPC 服务类
       // 因此继承实现了 BindableService 类但是没有使用 @GrpcServer 注解的类和使用了 @GrpcServer 注解的类但不是 BindableService 子类的 bean 都需要剔除掉
       List<String> beanNameList = new ArrayList<>();
       for (String name : beanNames) {
           if (beansWithAnnotation != null && beansWithAnnotation.containsKey(name)) {
               beanNameList.add(name);
           }
       }

       // 根据 beanName 获取这些 bean 实例，并注册服务
       for (String name : beanNameList) {
           BindableService bindableService = applicationContext.getBeanFactory().getBean(name, BindableService.class);
           serverBuilder.addService(bindableService);
           log.info("{} 服务已经注册", name);
       }

        // 启动服务类
        server = serverBuilder.build().start();
        log.info("gRPC 服务端已经启动, 端口号 : {}", grpcServerProperties.getPort());
        startDaemonAwaitThread();
    }

    private void startDaemonAwaitThread() {
        Thread awaitThread = new Thread() {
            @Override
            public void run() {
                try {
                    GrpcServerRunner.this.server.awaitTermination();
                } catch (InterruptedException e) {
                    log.error("启动 gRPC 服务出现异常", e);
                }
            }

        };
        awaitThread.setDaemon(false);
        awaitThread.start();
    }

    @Override
    public void destroy() throws Exception {
        log.info("关闭 gRPC 服务端 ...");
        Optional.ofNullable(server).ifPresent(Server::shutdown);
        log.info("gRPC 服务端已经关闭");
    }
}
```

下面的代码是添加了拦截器和优化过后的代码，也是在项目中实际使用的代码。

```java
@Slf4j
@Configuration
@EnableConfigurationProperties(value = {GrpcServerProperties.class})
public class GrpcServerRunner implements CommandLineRunner, DisposableBean  {

    private final AbstractApplicationContext applicationContext;
    private final GrpcServerProperties grpcServerProperties;
    private Server server;

    @Autowired
    public GrpcServerRunner(AbstractApplicationContext applicationContext, GrpcServerProperties grpcServerProperties) {
        this.applicationContext = applicationContext;
        this.grpcServerProperties = grpcServerProperties;
    }

    @Override
    public void run(String... args) throws Exception {
        log.info("开始创建 gRPC 服务 ...");

        // 拦截器
        Collection<ServerInterceptor> globalInterceptors = getBeanNamesByTypeWithAnnotation(GRpcGlobalInterceptor.class,ServerInterceptor.class)
                .map(name -> applicationContext.getBeanFactory().getBean(name,ServerInterceptor.class))
                .collect(Collectors.toList());

        final ServerBuilder<?> serverBuilder = ServerBuilder.forPort(grpcServerProperties.getPort());

        // 注册和绑定拦截器
        getBeanNamesByTypeWithAnnotation(GrpcService.class,BindableService.class)
                .forEach(name->{
                    BindableService srv = applicationContext.getBeanFactory().getBean(name, BindableService.class);
                    ServerServiceDefinition serviceDefinition = srv.bindService();
                    GrpcService gRpcServiceAnn = applicationContext.findAnnotationOnBean(name,GrpcService.class);
                    serviceDefinition  = bindInterceptors(serviceDefinition,gRpcServiceAnn,globalInterceptors);
                    serverBuilder.addService(serviceDefinition);
                    log.info("{} 服务已经注册", serviceDefinition.getServiceDescriptor().getName());

                });

        // 启动服务类
        server = serverBuilder.build().start();
        log.info("gRPC 服务端已经启动, 端口号 : {}", grpcServerProperties.getPort());
        startDaemonAwaitThread();
    }

    private void startDaemonAwaitThread() {
        Thread awaitThread = new Thread() {
            @Override
            public void run() {
                try {
                    GrpcServerRunner.this.server.awaitTermination();
                } catch (InterruptedException e) {
                    log.error("启动 gRPC 服务出现异常", e);
                }
            }

        };
        awaitThread.setDaemon(false);
        awaitThread.start();
    }

    @Override
    public void destroy() throws Exception {
        log.info("关闭 gRPC 服务端 ...");
        Optional.ofNullable(server).ifPresent(Server::shutdown);
        log.info("gRPC 服务端已经关闭");
    }

    private ServerServiceDefinition bindInterceptors(ServerServiceDefinition serviceDefinition, GrpcService gRpcService,
                                                     Collection<ServerInterceptor> globalInterceptors) {
        Stream<? extends ServerInterceptor> privateInterceptors = Stream.of(gRpcService.interceptors())
                .map(interceptorClass -> {
                    try {
                        return 0 < applicationContext.getBeanNamesForType(interceptorClass).length ?
                                applicationContext.getBean(interceptorClass) :
                                interceptorClass.newInstance();
                    } catch (Exception e) {
                        throw  new BeanCreationException("Failed to create interceptor instance.",e);
                    }
                });

        List<ServerInterceptor> interceptors = Stream.concat(
                    gRpcService.applyGlobalInterceptors() ? globalInterceptors.stream(): Stream.empty(),
                    privateInterceptors)
                .distinct()
                .collect(Collectors.toList());
        return ServerInterceptors.intercept(serviceDefinition, interceptors);
    }

    private <T> Stream<String> getBeanNamesByTypeWithAnnotation(Class<? extends Annotation> annotationType, Class<T> beanType) throws Exception{
       return Stream.of(applicationContext.getBeanNamesForType(beanType))
                .filter(name->{
                    final BeanDefinition beanDefinition = applicationContext.getBeanFactory().getBeanDefinition(name);
                    final Map<String, Object> beansWithAnnotation = applicationContext.getBeansWithAnnotation(annotationType);

                    if ( !beansWithAnnotation.isEmpty() ) {
                        return beansWithAnnotation.containsKey(name);
                    } else if( beanDefinition.getSource() instanceof StandardMethodMetadata) {
                        StandardMethodMetadata metadata = (StandardMethodMetadata) beanDefinition.getSource();
                        return metadata.isAnnotated(annotationType.getName());
                    }
                    return false;
                });
    }
}
```

# io.grpc.StatusRuntimeException: UNKNOWN 错误

etcd4j 和 gRPC 底层实现都使用了 Netty, 我这里出现这个错误是由于 Netty 版本冲突造成的, 当时 etcd4j 使用的版本是 2.15.0, gRPC 的版本是 1.7.0, 将 gRPC 的版本改成 1.13.1 就好了。

```text
Exception in thread "main" io.grpc.StatusRuntimeException: UNKNOWN
    at io.grpc.stub.ClientCalls.toStatusRuntimeException(ClientCalls.java:210)
    at io.grpc.stub.ClientCalls.getUnchecked(ClientCalls.java:191)
    at io.grpc.stub.ClientCalls.blockingUnaryCall(ClientCalls.java:124)
    at com.hello.grpc.proto.UserServiceGrpc$UserServiceBlockingStub.getUserInfo(UserServiceGrpc.java:160)
    at com.grpc.server.nospring.MainClient.main(MainClient.java:26)
Caused by: java.lang.IndexOutOfBoundsException: readerIndex(0) + length(10) exceeds writerIndex(0):
           PooledUnsafeDirectByteBuf(ridx: 0, widx: 0, cap: 30)
    at io.netty.buffer.AbstractByteBuf.checkReadableBytes0(AbstractByteBuf.java:1403)
    at io.netty.buffer.AbstractByteBuf.checkReadableBytes(AbstractByteBuf.java:1390)
    at io.netty.buffer.AbstractByteBuf.readSlice(AbstractByteBuf.java:856)
    ...
```