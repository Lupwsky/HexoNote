---
title: Java-IO和IO流
date: 2019-03-12 21:57:06
categories: Java
---

# I/O

I/O, 即输入和输出, 通常指数据在内部存储器和外部存储器或其他周边设备之间的输入和输出, 几乎任何涉及到计算机与其他设备之间的数据交换的都可称之为 I/O, 如内存和磁盘, 计算机之间网络数据交换等, 一个完整的 IO 操作指的是 I/O 的调用和 I/O 的执行两个部分, 首先由应用程序调用操作系统的 I/O 接口进行 I/O 操作, 随即操作系统底层执行相关的 I/O 的操作

# I/O 流

I/O 操作中, 一个重要的知识点是数据的交换, 这个数据交换的过程就是数据的输入和数据的输出, 在 Java 的 I/O 操作中, 对数据的输入输出操作都是以 "流" 的方式进行的, 即 I/O 流, 通常也称之为数据流, "流" 是一个抽象的概念，它是对输入输出设备的一种抽象理解, 表示数据从源对象 (如文件, 内存, 网络) 按顺序流向目标对象的一种流动形式, 例如在 Java 中应用程序需要向文件中写数据的时候, 就需要开启一个输出流, 然后往输出流中写入数据, 反之则需要开启一个输入流, 然后从输入流中接收数据

<!-- more -->

# I/O 操作类的分类

[深入分析 Java I/O 的工作机制](https://www.ibm.com/developerworks/cn/java/j-lo-javaio/index.html) 一文的分类如下, 个人很赞同作者的这种分类, 所以直接贴出了该文中分类:

基于字节操作的 I/O 接口: InputStream 和 OutputStream
基于字符操作的 I/O 接口: Writer 和 Reader
基于磁盘操作的 I/O 接口: File
基于网络操作的 I/O 接口: Socket (注意 Socket 类并不在 java.io 包下)

# 使用示例

创建一个 `test.txt`文件, 输入内容: `测试读取数据`, 从文件中读取数据, 示例代码如下:

```java
FileInputStream fileInputStream = null;
try {
    fileInputStream = new FileInputStream("D:\\Work\\hello-grpc\\java-guava\\src\\main\\resources\\md\\test.txt");
    StringBuilder stringBuilder = new StringBuilder();

    // 每次最多读取 1024 个字节
    byte[] bytes = new byte[1024];
    while (fileInputStream.read(bytes) != -1) {
        // 字节转字符串
        stringBuilder.append(new String(bytes));
    }

    String value = stringBuilder.toString();
    log.info("length = {}, value = {}", value.length(), value);
} catch (IOException e) {
    e.printStackTrace();
} finally {
    if (fileInputStream != null) {
        try {
            // 关闭流
            fileInputStream.close();
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}
```

输出结果如下:

```text
length = 1012, value = 测试读取数据
```

`长度不应该为 6 么?` 由于使用了 `byte[] bytes = new byte[1024]` 保存字节, 实际上文件中只有 `测试读取数据` 个字符, bytes 数组没有内容的区域默认值为 0, 在 `stringBuilder.append(new String(bytes))` 转换的时候也被转换了, 字符长度是 1012, 为什么是 1012 而不是 1024 呢? 如果全部为 0, 则刚好是 1024 个字符长度, 但是系统默认使用的编码格式是 UTF-8, 汉字 UTF-8 占用 3 个字节或 4 个字节, 这里的 6 个汉字都是占用 3 个字节, 6 个字符占用了 18 个字节, 剩下的字节 1006 转换成 1006 个字符, 所以一共 1012 个长度的字符, 但是 StringBuilder 里面的值有一个变量记录的这个长度, 在使用 StringBuilder.toString(), 将 len 值直接给了 String 对象, 又由于将 bytes 数组在转化成字符串的时候丢弃了全部为 0 的字节, 所以最终输出的 value 长度为 1012 的, value 的值为 `测试读取数据` 的结果 

解决方法可以定义一个 len 记录每次读取字节数的长度, 字节转字符串时指定转换的字节长度, 示例代码如下:

```java
// 记录长度
int len;
// 每次最多读取 1024 个字节
byte[] bytes = new byte[1024];
while ((len = fileInputStream.read(bytes)) != -1) {
    // 字节转字符串
    stringBuilder.append(new String(bytes, 0, len));
}
```

或者`使用 read() 方法, 每次都一个字节 (效率比都多个字节底), 但是这种方式只能读取英文字符和数字, 读取中文会有问题`, 示例代码如下:

```java
int readValue;
while ((readValue = fileInputStream.read()) != -1) {
    // 字节转字符串
    stringBuilder.append((byte) readValue);
}
```

输出结果如下:

```text
length = 54, value = 230181139232175149232175187229143150230149176230141174
```

一般读取文件内容, 一般使用字符流读取, 可以解决读取中文的问题, 创建字符流的时候还能指定字符集, 这样就能解决中文乱码的问题, 示例代码如下:

```java
FileInputStream fileInputStream = null;
fileInputStream = new FileInputStream("D:\\Work\\hello-grpc\\java-guava\\src\\main\\resources\\md\\test.txt");
// 字节输入流转换成字符输入流, 并指定编码集, 不指定默认使用 UTF-8
InputStreamReader inputStreamReader = new InputStreamReader(fileInputStream, StandardCharsets.UTF_8);
// 加入缓存提高读取效率, 每次读取一行
BufferedReader bufferedReader = new BufferedReader(inputStreamReader);

StringBuilder stringBuilder = new StringBuilder();
String readValue;
while ((readValue = bufferedReader.readLine()) != null) {
    stringBuilder.append(readValue);
}
String value = stringBuilder.toString();
log.info("length = {}, value = {}", value.length(), value);
```

不加入缓存的处理方式, 和字节的读取方式一样, 需要设置一个 len 记录读取的长度:

```java
FileInputStream fileInputStream = null;
fileInputStream = new FileInputStream("D:\\Work\\hello-grpc\\java-guava\\src\\main\\resources\\md\\test.txt");
InputStreamReader inputStreamReader = new InputStreamReader(fileInputStream, StandardCharsets.UTF_8);

char[] chars = new char[1024];
StringBuilder stringBuilder = new StringBuilder();
int len;
while ((len = inputStreamReader.read(chars)) != -1) {
    stringBuilder.append(new String(chars, 0, len));
}
```

输出结果如下:

```text
length = 6, value = 测试读取数据
```

# 参考资料

* [深入理解 Java 流机制 (一)](https://www.cnblogs.com/forget406/p/5316452.html)
* [深入分析 Java I/O 的工作机制](https://www.ibm.com/developerworks/cn/java/j-lo-javaio/index.html)
* [Java I/O 总结](http://www.importnew.com/23708.html) - 如何选择I/O流
* [Java I/O 流学习总结一 : 输入输出流](https://blog.csdn.net/zhaoyanjun6/article/details/54292148)
