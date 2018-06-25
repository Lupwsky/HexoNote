---
title: SpringBoot-gRPC的使用
date: 2018-06-21 23:44:45
categories: Spring
---

上一篇笔记中，使用了 SpringBoot 自动配置的功能在 SpringBoot 项目中自动启动 gRPC 服务和客户端连接到指定的服务，我们要做的仅仅是在 application.properties 为文件中配置服务端的 port 和客户端需要连接的 host 和 port，主要是在 SpringBoot 项目中引入下面的两个依赖，这两个依赖对 gRPC 的 API 进行了封装，在项目启动后自动的完成服务端的启动和客户端的连接等初始化工作。

```xml
<properties>
    <spring.boot.grpc.server.version>1.4.0.RELEASE</spring.boot.grpc.server.version>
</properties>

<!-- 客户端 -->
<dependency>
    <groupId>net.devh</groupId>
    <artifactId>grpc-client-spring-boot-starter</artifactId>
    <version>${spring.boot.grpc.client.version}</version>
</dependency>

<!-- 服务端 -->
<dependency>
    <groupId>net.devh</groupId>
    <artifactId>grpc-server-spring-boot-starter</artifactId>
    <version>${spring.boot.grpc.server.version}</version>
</dependency>
```

<!-- more -->

今天这篇笔记主要记录了 gRPC 没有封装时是如何使用的。

# 引入依赖

新建一个 Maven 管理的 JavaWeb 项目，添加以下依赖。

```xml
<properties>
    <grpc.version>1.10.0</grpc.version>
    <protobuf.version>3.3.0</protobuf.version>
    <os.plugin.version>1.5.0.Final</os.plugin.version>
    <protobuf.maven.plugin.version>0.5.0</protobuf.maven.plugin.version>
</properties>

<dependencies>
    <dependency>
        <groupId>io.grpc</groupId>
        <artifactId>grpc-netty</artifactId>
        <version>${grpc.version}</version>
    </dependency>

    <dependency>
        <groupId>io.grpc</groupId>
        <artifactId>grpc-protobuf</artifactId>
        <version>${grpc.version}</version>
    </dependency>

    <dependency>
        <groupId>io.grpc</groupId>
        <artifactId>grpc-stub</artifactId>
        <version>${grpc.version}</version>
    </dependency>

    <dependency>
        <groupId>com.google.protobuf</groupId>
        <artifactId>protobuf-java</artifactId>
        <version>${protobuf.version}</version>
    </dependency>
</dependencies>

<build>
    <finalName>grpc-proto</finalName>
    <extensions>
        <extension>
            <groupId>kr.motd.maven</groupId>
            <artifactId>os-maven-plugin</artifactId>
            <version>${os.plugin.version}</version>
        </extension>
    </extensions>

    <plugins>
        <plugin>
            <groupId>org.xolstice.maven.plugins</groupId>
            <artifactId>protobuf-maven-plugin</artifactId>
            <version>${protobuf.maven.plugin.version}</version>
            <configuration>
                <protocArtifact>com.google.protobuf:protoc:${protobuf.version}:exe:${os.detected.classifier}</protocArtifact>
                <pluginId>grpc-java</pluginId>
                <pluginArtifact>io.grpc:protoc-gen-grpc-java:${grpc.version}:exe:${os.detected.classifier}</pluginArtifact>
            </configuration>
            <executions>
                <execution>
                    <goals>
                        <goal>compile</goal>
                        <goal>compile-custom</goal>
                    </goals>
                </execution>
            </executions>
        </plugin>
    </plugins>
</build>
```

# 编写 protobuf 文件

编写测试用的 protobuf 问价，然后编译生成客户端和服务端需要的接口和类，操作见上一篇笔记。

```proto
syntax = "proto3";

option java_multiple_files = true;
option java_package = "com.hello.grpc.proto";
option java_outer_classname = "TestServiceProto";

// 定义的 RPC 接口
service TestService {
    // 可被调用的方法
    rpc test (Request) returns (Response){}
}

message Request {
    int32 id = 1;
}

message Response {
    int32 id = 1;
    string name = 2;
}
```

# 创建服务端

```java
public class GrpcServer {
    /**
     * 服务绑定的端口号
     */
    private int port;

    private Server server;

    /**
     * 构造函数，创建并启动服务
     * 
     * @param port 端口号
     * @param bindableService 绑定的服务
     * @param serverInterceptor
     */
    public GrpcServer(int port, BindableService bindableService) {
        this.port = port;
        start(bindableService, serverInterceptor);
    }

    /**
     * 创建并启动 gRPC 服务
     *
     * @param bindableService 服务接口实现
     */
    private void start(BindableService bindableService) {
        try {
            // 可以添加绑定多个服务接口实现
            server = ServerBuilder.forPort(port).addService(bindableService).build().start();
            System.out.println("gRPC 服务已启动!");

            // JVM 停止运行的时候关闭 gRPC
            Runtime.getRuntime().addShutdownHook(new Thread() {
                @Override
                public synchronized void start() {
                    if (server != null) {
                        server.shutdown();
                    }
                }
            });
        } catch (IOException e) {
            e.printStackTrace();
        }
    }

    /**
     * 停止 gRPC 服务
     *
     */
    private void stop() {
        if (server != null) {
            server.shutdown();
        }
        System.out.println("gRPC 服务已关闭!");
    }

    /**
     * 阻塞，一直到程序退出运行
     *
     * @throws InterruptedException
     */
    public void blockUntilShutdown() {
        if (server != null) {
            try {
                server.awaitTermination();
            } catch (InterruptedException e) {
                e.printStackTrace();
            } finally {
                stop();
            }
        }
    }
}
```

# 服务实现类

创建 TestServiceGrpcImpl 类实现生成的 TestServiceGrpc.TestServiceImplBase 接口类

```java
public class TestServiceGrpcImpl extends TestServiceGrpc.TestServiceImplBase {

    @Override
    public void test(Request request, StreamObserver<Response> responseObserver) {
        int id = request.getId();
        responseObserver.onNext(Response.newBuilder().setId(id).setName("test").build());
        responseObserver.onCompleted();
    }
}
```

# 启动服务端

```java
public class MainServer {
    public static void main(String[] args) {
        TestServiceGrpcImpl testServiceGrpc = new TestServiceGrpcImpl();
        GrpcServerInterceptor serverInterceptor = new GrpcServerInterceptor();
        GrpcServer server = new GrpcServer(8000, testServiceGrpc, serverInterceptor);
        server.blockUntilShutdown();
        System.out.println("进程结束!");
    }
}
```

# 启动客户端进行连接并调用RPC方法

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

运行 MainClient 的 main 方法后，控制台会打印 id 和 name，已经成功的调用了服务提供的方法。

