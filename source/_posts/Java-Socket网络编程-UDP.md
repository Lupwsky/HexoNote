---
title: Java-Socket网络编程-UDP
date: 2019-03-12 22:11:58
categories: Java
---

UDP (用户数据保协议), 是一种面向无连接的传输层协议, 和 TCP 不同的是, UDP 是不可靠的 (是指该协议在网络环境不好的情况下, 会丢失数据包), 也不能像 TCP 那样对数据包进行分组, 组装, 排序和重传, 使用 UDP 也不能知道发送的数据包时候安全完整的传输到目的地, 因为它是无连接型协议, 因而具有资源消耗小, 处理速度快的优点, 通常音频, 视频和普通数据在传送时使用 UDP 较多, 因为它们即使偶尔丢失一两个数据包, 也不会对接收结果产生太大影响, 如视频聊天时, 丢失某些帧对聊天效果影响不大, 同时由于排除了信息可靠传递机制, 将安全和排序等功能移交给上层应用来完成, 因此极大地减少了执行时间, 使速度得到了保证

# 服务端简单示例

Java 提供了 DatagramSocket 类来实现 UDP 的通信, 传输的数据需要封装在 DatagramPacket 类中, UDP 中没有明确的服务端和客户端的划分, 是一个相对的, 要看是接收数据还是发送数据, 先来看看如何使用的, 让服务端监听端口 9000, 不停的接收来自客户端的数据, 示例代码如下:

```java
public static void main(String[] args) throws IOException {
    // 监听 9000 端口, 接收发送到 9000 端口的数据
    DatagramSocket datagramSocket = new DatagramSocket(9000);
    log.info("服务端绑定端口号 9000 成功");

    // 接收数据, DatagramPacket 的 length 参数要和客户端的一样
    byte[] byteArray = new byte[1024];
    while (true) {
        log.info("服务端正在接收数据");
        DatagramPacket datagramPacket = new DatagramPacket(byteArray, 1024);
        datagramSocket.receive(datagramPacket);
        String content = new String(datagramPacket.getData(), 0, datagramPacket.getLength());
        log.info("收到客户端的消息 = {}", content);
    }

    // TODO 关闭资源
}
```

<!-- more -->

# 客户端简单示例

客户端监听端口 9001, 并且不停的向 9000 端口发送数据, 示例代码如下:

```java
public static void main(String[] args) throws IOException, InterruptedException {
    // 客户端监听端口号 9001, 接收发送到 9001 端口的数据
    DatagramSocket datagramSocket = new DatagramSocket(9001);
    log.info("客户端绑定端口号 9001 成功");

    // 将数据发送到指定主机和指定端口的服务端上
    while (true) {
        Thread.sleep(1000);
        String content = "来自客户端的问候, time = " + DateTime.now().toString("yyyy-MM-dd HH:mm:ss");
        DatagramPacket datagramPacket = new DatagramPacket(content.getBytes(), content.getBytes().length, InetAddress.getLocalHost(), 9000);
        datagramSocket.send(datagramPacket);
        log.info("客户端发送消息成功, content = " + content);
    }

    // TODO 关闭资源
}
```

# 测试结果

服务端输出结果如下:

```text
23:05:05.241 [main] - 服务端绑定端口号 9000 成功
23:05:05.244 [main] - 服务端正在接收数据
23:05:29.585 [main] - 收到客户端的消息 = 来自客户端的问候, time = 2019-03-13 23:05:29
23:05:29.590 [main] - 服务端正在接收数据
23:05:30.589 [main] - 收到客户端的消息 = 来自客户端的问候, time = 2019-03-13 23:05:30
23:05:30.590 [main] - 服务端正在接收数据
23:05:31.595 [main] - 收到客户端的消息 = 来自客户端的问候, time = 2019-03-13 23:05:31
23:05:31.595 [main] - 服务端正在接收数据
23:05:32.600 [main] - 收到客户端的消息 = 来自客户端的问候, time = 2019-03-13 23:05:32
```

客户端输出结果如下:

```text
23:05:28.482 [main] - 客户端绑定端口号 9001 成功
23:05:29.584 [main] - 客户端发送消息成功, content = 来自客户端的问候, time = 2019-03-13 23:05:29
23:05:30.589 [main] - 客户端发送消息成功, content = 来自客户端的问候, time = 2019-03-13 23:05:30
23:05:31.595 [main] - 客户端发送消息成功, content = 来自客户端的问候, time = 2019-03-13 23:05:31
23:05:32.600 [main] - 客户端发送消息成功, content = 来自客户端的问候, time = 2019-03-13 23:05:32
```

# UDP 数据包的大小

一个 UDP 包最大的长度为 2 ^ 16 - 1 (65536 - 1 = 65535), 最大的发送长度为65535, 在这65535之内包含 IP 协议头的 20 个字节, 还有 UDP 协议头的 8 个字节, 即 65535 - 20 - 8 = 65507, 因此, UDP 传输用户数据最大的长度为65507, 超过这个数值则会抛出异常

# UDP 实现广播

实现广播需要客户端设置调用方法 `setBroadcast()` 实现, 将上文中的客户端改造如下:

```java
DatagramSocket datagramSocket = new DatagramSocket(9001);
// 设置为广播, 并发送数据到 9000 端口
datagramSocket.setBroadcast(true);
while (true) {
    Thread.sleep(1000);
    String content = "来自客户端的问候, time = " + DateTime.now().toString("yyyy-MM-dd HH:mm:ss");
    DatagramPacket datagramPacket = new DatagramPacket(content.getBytes(), content.getBytes().length, InetAddress.getLocalHost(), 9000);
    datagramSocket.send(datagramPacket);
}
```

接着创建两个或者多个服务端, 监听来自 9000 端口的数据, 代码和上面的 `服务端简单示例` 中的代码一样, 不需要改动, 拷贝几份即可, 启动后, 客户端发送的消息, 服务端都可以收到消息

# UDP 实现组播

实现组播需要使用 MulticastSocket 类实现, 创建多个服务端, 使用 joinGroup() 方法加入同一个组播, 代码如下:

```java
// 服务端一
MulticastSocket multicastSocket = new MulticastSocket(9000);
multicastSocket.joinGroup(InetAddress.getByName("127.0.0.1"));
byte[] byteArray = new byte[1024];
while (true) {
    DatagramPacket datagramPacket = new DatagramPacket(byteArray, 1024);
    multicastSocket.receive(datagramPacket);
    String content = new String(datagramPacket.getData(), 0, datagramPacket.getLength());
    log.info("收到客户端的消息 = {}", content);
}

// 服务端二
MulticastSocket multicastSocket = new MulticastSocket(9000);
multicastSocket.joinGroup(InetAddress.getLocalHost());
byte[] byteArray = new byte[1024];
while (true) {
    DatagramPacket datagramPacket = new DatagramPacket(byteArray, 1024);
    multicastSocket.receive(datagramPacket);
    String content = new String(datagramPacket.getData(), 0, datagramPacket.getLength());
    log.info("收到客户端的消息 = {}", content);
}
```

客户端代码如下:

```java
MulticastSocket multicastSocket = new MulticastSocket(9001);
while (true) {
    Thread.sleep(1000);
    String content = "来自客户端的问候, time = " + DateTime.now().toString("yyyy-MM-dd HH:mm:ss");
    DatagramPacket datagramPacket = new DatagramPacket(content.getBytes(), content.getBytes().length, InetAddress.getLocalHost(), 9000);
    datagramSocket.send(datagramPacket);
    log.info("客户端发送消息成功, content = " + content);
}
```

# 参考资料

* NIO 与 SOCKET 编程技术指南 - 书籍
* [A Guide To UDP In Java](https://www.baeldung.com/udp-in-java)
* [Java 网络编程 TCP 和 UDP](https://blog.xiaoxiaomo.com/2016/04/12/Java-%E7%BD%91%E7%BB%9C%E7%BC%96%E7%A8%8BTCP%E5%92%8CUDP/)