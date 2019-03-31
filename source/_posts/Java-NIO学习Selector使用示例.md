---
title: Java-NIO学习Selector使用示例
date: 2019-03-15 23:18:44
categories: Java
---

NIO 使用 ServerSocketChannel 和 SocketChannel 可以实现基于 TCP 协议的 Socket 通信, 但是如果没有将通道和选择器结合起来使用, 实现的 Scoket 通信和 BIO 实现的方式是一样的模式, 没有实现多路复用, 这里记录下结合 Selector 实现 Socket 通信多路复用的功能, 一些知识点和注意的地方就直接写在注释里面了, 不想整理了~~

# 服务端

```java
public static void main(String[] args) throws IOException, InterruptedException {
    ServerSocketChannel serverSocketChannel = ServerSocketChannel.open();

    // Channel 与 Selector 一起使用实现 I/0 多路复用，Channel 必须处于非阻塞模式下
    serverSocketChannel.configureBlocking(false);

    // 绑定端口号
    ServerSocket serverSocket = serverSocketChannel.socket();
    serverSocket.bind(new InetSocketAddress(9000));

    // 将 Channel 注册到选择器上, 第二个参数表示需要监听的 Channel 操作事件, 多个监听操作事件使用位或操作实现, 监听事操作件的类型如下
    // SelectionKey.OP_ACCEPT, 表示服务器接受客户端连接的监听事件
    // SelectionKey.OP_CONNECT, 表示客户端连接到服务器的监听事件
    // SelectionKey.OP_READ, 表示服务器准备从通道中读取数据的监听事件
    // SelectionKey.OP_WRITE, 表示服务器准备写入数据到通道的监听事件
    // 另外要注意的是这些事件只有一次是有效的

    // 创建选择器实例, 将服务端注册到选择器, 监听 OP_ACCEPT 事件
    // SelectionKey 代表了注册到该 Selector 的 Channel, 可以从这里面获取到对应的 Channel 信息和绑定的 Selector 信息, 如监听了哪些事件
    // 服务端只能支持 OP_ACCEPT 监听事件, 使用 ServerSocketChannel.validOps() 可以知道对应的 Channel 支持的监听事件
    Selector selector = Selector.open();
    SelectionKey selectionKey = serverSocketChannel.register(selector, SelectionKey.OP_ACCEPT);

    // 调用 Selector.select() 方法, 返回发生对应监听事件的那些 Channel, 返回 int 表示有多少个 Channel 就绪了
    // select() 阻塞到至少有一个通道在你注册的事件上就绪了
    // select(long timeout) 阻塞到至少有一个通道在你注册的事件上就绪了, 设置最大阻塞时间
    // selectNow() 不阻塞, 如果没有 Channel 就绪, 返回 0, 这种方式需要不停循环检测
    while (true) {
        if (selector.selectNow() == 0) {
            Thread.sleep(1000);
            continue;
        }

        // 获取已经准备就绪的 SelectionKey, 里面带有 Channel 信息和是那种操作事件已经就绪
        Iterator<SelectionKey> keyIterator = selector.selectedKeys().iterator();
        while (keyIterator.hasNext()) {
            SelectionKey tempSelectionKey = keyIterator.next();

            // 表示有客户端连接上来, 可以从 SelectionKey 取到客户端的 SeverSocketChannel 实例, 并注册到选择器
            // 通过 SeverSocketChannel.accept() 获取到客户端的 SocketChannel 后注册到选择器, 并将
            if (tempSelectionKey.isAcceptable()) {
                log.info("有客户端连接上来了, 开始接收数据");
                SocketChannel socketChannel = ((ServerSocketChannel) tempSelectionKey.channel()).accept();
                socketChannel.configureBlocking(false);
                socketChannel.register(tempSelectionKey.selector(), SelectionKey.OP_READ | SelectionKey.OP_WRITE);
            }

            // 客户端发送数据过来时, 可以开始读取客户端发送的数据了
            if (tempSelectionKey.isValid() && tempSelectionKey.isReadable()) {
                SocketChannel socketChannel = (SocketChannel) tempSelectionKey.channel();
                log.info(receiveMsg(socketChannel));

                // 发送消息到客户端
                String content = "这是服务端发送的数据";
                log.info("服务端发送消息 = {}", content);
                sendMsg(socketChannel, content);
            }

            // SelectionKey.isWritable() 是表示 SocketChannel 可写, 网络不出现阻塞情况下, 一直是可以写的, 所认一直为 true, 一般我们不注册 OP_WRITE
            // 如果上面 socketChannel.register(tempSelectionKey.selector(), SelectionKey.OP_READ | SelectionKey.OP_WRITE);
            // 这里会不停的打印 "SocketChannel 可以写入数据了", 实际项目开发中这样会造成 CPU 极大的浪费
            if (tempSelectionKey.isValid() && tempSelectionKey.isWritable()) {
                log.info("SocketChannel 可以写入数据了");
            }

            // Selector 不会自己从已选择键集中移除 SelectionKey 实例, 必须在处理完通道时自己移除
            // 下次该通道变成就绪时, Selector 会再次将其放入集合中
            keyIterator.remove();
        }
    }
}
```

<!-- more -->

服务端发送数据和接收数据示例代码:

```java
// 发送数据
private static void sendMsg(SocketChannel socketChannel, String msg) {
    try {
        socketChannel.write(ByteBuffer.wrap(msg.getBytes()));
    } catch (IOException e) {
        log.info("服务端发送消息失败, error = {}", e);
    }
}

// 接收数据
private static String receiveMsg(SocketChannel socketChannel) {
    StringBuilder contentBuffer = new StringBuilder();
    try {
        ByteBuffer buffer = ByteBuffer.allocate(1024);
        int count;
        while((count = socketChannel.read(buffer)) > 0){
            contentBuffer.append(new String(buffer.array(), 0, count));
            buffer.rewind();
        }
    } catch (IOException e) {
        log.info("服务器接收消息失败 = {}, error = {}", e);
        try {
            socketChannel.close();
        } catch (IOException e1) {
            e1.printStackTrace();
        }
    }
    return contentBuffer.toString();
}
```

服务端输出结果如下:

```text
23:26:35.207 [main] INFO com.lupw.guava.io.NioServerSelector - 有客户端连接上来了, 开始接收数据
23:26:38.230 [main] INFO com.lupw.guava.io.NioServerSelector - 你好, 服务器2019-03-15 23:26:37
23:26:38.230 [main] INFO com.lupw.guava.io.NioServerSelector - 服务端发送消息 = 这是服务端发送的数据
23:26:41.244 [main] INFO com.lupw.guava.io.NioServerSelector - 你好, 服务器2019-03-15 23:26:40
23:26:41.244 [main] INFO com.lupw.guava.io.NioServerSelector - 服务端发送消息 = 这是服务端发送的数据
23:26:44.257 [main] INFO com.lupw.guava.io.NioServerSelector - 你好, 服务器2019-03-15 23:26:43
23:26:44.257 [main] INFO com.lupw.guava.io.NioServerSelector - 服务端发送消息 = 这是服务端发送的数据
```

# 客户端

```java
public static void main(String[] args) throws IOException {
    SocketChannel socketChannel = SocketChannel.open();
    socketChannel.connect(new InetSocketAddress("127.0.0.1", 9000));
    socketChannel.configureBlocking(false);

    new Thread(() -> {
        while (true) {
            try {
                Thread.sleep(3000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            String content = "你好, 服务器" + DateTime.now().toString("yyyy-MM-dd HH:mm:ss");
            log.info("客户端发送消息 = {}", content);
            sendMsg(socketChannel, content);
        }
    }).start();

    new Thread(() -> {
        while (true) {
            try {
                Thread.sleep(3000);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            log.info("客户端收到消息" + receiveMsg(socketChannel));
        }
    }
    ).start();
}
```

客户端发送数据和接收数据示例代码:

```java
// 发送数据
private static void sendMsg(SocketChannel socketChannel, String msg){
    try {
        socketChannel.write(ByteBuffer.wrap(msg.getBytes()));
    } catch (IOException e) {
        log.info("客户端发送消息失败, error = {}", e);
    }
}

// 接收数据
private static String receiveMsg(SocketChannel socketChannel){
    StringBuilder contentBuffer = new StringBuilder();
    try {
        ByteBuffer buffer = ByteBuffer.allocate(1024);
        int count;
        while((count = socketChannel.read(buffer)) > 0){
            contentBuffer.append(new String(buffer.array(), 0, count));
        }
    } catch (IOException e) {
        log.info("客户端接收消息失败 = {}, error = {}", e);
        try {
            socketChannel.close();
        } catch (IOException e1) {
            e1.printStackTrace();
        }
    }
    return contentBuffer.toString();
}
```

客户端输出结果如下:

```text
23:26:37.637 [Thread-1] INFO com.lupw.guava.io.NioClientSelector - 客户端收到消息
23:26:37.694 [Thread-0] INFO com.lupw.guava.io.NioClientSelector - 客户端发送消息 = 你好, 服务器2019-03-15 23:26:37
23:26:40.644 [Thread-1] INFO com.lupw.guava.io.NioClientSelector - 客户端收到消息这是服务端发送的数据
23:26:40.701 [Thread-0] INFO com.lupw.guava.io.NioClientSelector - 客户端发送消息 = 你好, 服务器2019-03-15 23:26:40
23:26:43.645 [Thread-1] INFO com.lupw.guava.io.NioClientSelector - 客户端收到消息这是服务端发送的数据
23:26:43.702 [Thread-0] INFO com.lupw.guava.io.NioClientSelector - 客户端发送消息 = 你好, 服务器2019-03-15 23:26:43
```

# 参考资料

* [NIO Selector 详解](https://segmentfault.com/a/1190000006824196)
* [Java NIO Selector详解 (含多人聊天室实例)](https://blog.csdn.net/jeffleo/article/details/54695959)