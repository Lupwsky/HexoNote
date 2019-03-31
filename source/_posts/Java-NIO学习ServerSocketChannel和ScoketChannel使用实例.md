---
title: Java-NIO学习ServerSocketChannel和ScoketChannel使用实例
date: 2019-03-15 23:10:25
categories: Java
---

在 NIO 中 提供了 ServerSocketChannel 和 ScoketChannel 两个类实现基于 TCP 协议的 Socket 通信, 使用示例记录如下

# 服务端实现

```java
private static void serverSocketChannel() throws IOException, InterruptedException {
    // 创建 ServerSocketChannel 实例
    ServerSocketChannel serverSocketChannel = ServerSocketChannel.open();

    // I/O 处理设置阻塞, 默认的方式也是阻塞, 多路复用中必须设置为 false
    serverSocketChannel.configureBlocking(true);

    // 创建 ServerSocket  实例
    ServerSocket serverSocket = serverSocketChannel.socket();
    serverSocket.bind(new InetSocketAddress(9000));

    log.info("服务端等待客户端连接...");
    SocketChannel socketChannel = serverSocketChannel.accept();

    if (socketChannel != null) {
        log.info("连接成功, 开始读取数据...");
        ByteBuffer readByteBuffer = ByteBuffer.allocate(1024);
        int len;
        while ((len = socketChannel.read(readByteBuffer)) != -1) {
            log.info("服务端收到到数据 = {}", new String(readByteBuffer.array(), 0, len));
            readByteBuffer.rewind();
        }
    } else {
        log.info("服务端等待客户端连接失败");
    }

    // TODO 关闭资源
}
```

<!-- more -->

直接使用 ServerSocketChannel, `不将 Channel 注册到选择器实现的 Socket 通信和普通的 BIO 实现是一样的`, 不同的是, ServerSocketChannel, SocketChannel 都可以设置为非阻塞模式, 调用 configureBlocking(), 参数设置为 false 即可, 当设置成非阻塞模式时, 如果没有客户端连接进来, 返回 null, 需要循环判断是否有客户端连接, 如下:

```java
while (true) {
    // 非阻塞模式, accept() 方法不会阻塞
    SocketChannel channel = serverSocketChannel.accept();
    if (channel != null) {
        Thread.sleep(1000);
        log.info("过一段时间再去看看是否有客户端连接上了");
    } else {
        // TODO do something
    }
}
```

# 客户端实现

```java
private static void socketChannel() throws IOException, InterruptedException {
    SocketChannel socketChannel = SocketChannel.open();
    socketChannel.configureBlocking(true);

    log.info("开始连接到服务端...");
    // 开启了阻塞模式, 这里在连接的会阻塞知道连接成功或者失败
    boolean connectResult = socketChannel.connect(new InetSocketAddress("127.0.0.1", 9000));
    if (connectResult) {
        log.info("连接到服务端成功");

        while (true) {
            Thread.sleep(1000);
            ByteBuffer byteBuffer = ByteBuffer.allocate(1024);
            String content = "发送当前时间吧, " + DateTime.now().toString("yyyy-MM-dd HH:mm:ss:SSS");
            log.info("开始发送消息 = {}", content);
            byteBuffer.put(content.getBytes());
            byteBuffer.flip();

            socketChannel.write(byteBuffer);
            byteBuffer.rewind();
        }
    } else {
        log.info("连接到服务端失败");
    }

    // TODO 关闭资源
}
```

非阻塞模式需要循环判断连接到服务器是否成功, 如下:

```java
while (true) {
    // 非阻塞模式, connect() 方法不会阻塞
    boolean result = socketChannel.connect(new InetSocketAddress("127.0.0.1", 9000));
    // 或者调用 socketChannel.finishConnect() 来判断是否连接成功
    if (!result) {
        Thread.sleep(1000);
        log.info("等会再看看是否连接到服务器成功");
    } else {
        // TODO do something
    }
}
```