---
title: Java-NIO学习DatagramChannel使用示例
date: 2019-03-15 23:04:31
categories: Java
---

在 NIO 中 DatagramChannel 通道可实现 UDP 协议的 Socket 通信, 记录下学习时的测试代码, 如下

# 服务器 A 发送和接收报文示例

```java
private static void nioDatagramChannelA() throws IOException {
    DatagramChannel datagramChannel = DatagramChannel.open();
    datagramChannel.configureBlocking(true);

    // 监听来自 9000 端口的数据
    DatagramSocket datagramSocket = datagramChannel.socket();
    datagramSocket.bind(new InetSocketAddress(9000));

    log.info("A 等待 B 发送的消息...");
    ByteBuffer readByteBuffer = ByteBuffer.allocate(1024);
    datagramChannel.receive(readByteBuffer);
    readByteBuffer.flip();
    log.info("A 收到 B 发送的消息 = {}", new String(readByteBuffer.array()));

    // 向 9001 端口发送数据
    String content = "A 发来的消息";
    ByteBuffer writeByteBuffer = ByteBuffer.allocate(1024);
    writeByteBuffer.put(content.getBytes());
    writeByteBuffer.flip();
    datagramChannel.send(writeByteBuffer, new InetSocketAddress("127.0.0.1", 9001));
}
```

<!-- more -->

注: 可以将 DatagramChannel "连接" 到网络中的特定地址的, 由于 UDP 是无连接的, 连接到特定地址并不会像 TCP 通道那样创建一个真正的连接, 而是锁住 DatagramChannel ，让其只能从特定地址收发数据, 如下代码所示:

```java
channel.connect(new InetSocketAddress("192.168.1.199", 8080));
```

注: DatagramChannel 连接成功后, 也可以使用 read() 和 write() 方法, 就像在用传统的通道一样, 只是在数据传送方面没有任何保证

# 服务器 B 发送和接收报文示例

```java
private static void nioDatagramChannelB() throws IOException {
    DatagramChannel datagramChannel = DatagramChannel.open();
    datagramChannel.configureBlocking(true);

    // 监听来自 9001 端口的数据
    DatagramSocket datagramSocket = datagramChannel.socket();
    datagramSocket.bind(new InetSocketAddress(9001));

    // 向 9001 端口发送数据
    String content = "B 发来的消息";
    ByteBuffer writeByteBuffer = ByteBuffer.allocate(1024);
    writeByteBuffer.put(content.getBytes());
    writeByteBuffer.flip();
    datagramChannel.send(writeByteBuffer, new InetSocketAddress("127.0.0.1", 9000));

    // 接收消息
    log.info("B 等待 A 发送的消息...");
    ByteBuffer readByteBuffer = ByteBuffer.allocate(1024);
    datagramChannel.receive(readByteBuffer);
    readByteBuffer.flip();
    log.info("B 收到 A 发送的消息 = {}", new String(readByteBuffer.array()));
}
```