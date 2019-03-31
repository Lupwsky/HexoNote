---
title: Java-NIO学习缓存API的使用
date: 2019-03-10 22:48:01
categories: Java
---

最近开始在看 `NIO于Scoket编程技术指南` 一书, 记录下学习笔记

# NIO 中的缓存类 API 使用

和传统 I/O 只能在处理 byte 和 char 型数据, NIO 支持多种类型的数据 (底层还是 byte 字节来处理), NIO 使用 Buffer 类处理不同的的数据类型, Buffer 是一个抽象类, 有 7 个直接的之类: ByteBuffer, CharBuffer, DoubleBuffer, FloatBuffer, IntBuffer, LongBuffer, ShortBuffer, 注意这 7 个类也是抽象类, 不能直接实例化, 但是这些类都提供了一个 wrap 方法将数据放入缓存中, 并获取实例:

* 实例化方法一, 使用已经存在的数组: ByteBuffer byteBuffer = ByteBuffer.wrap(new byte[]{'1', '2', '3', '4', '5'});
* 实例化方法二, 使用 allocate 方法: ByteBuffer byteBuffer = ByteBuffer.allocate(10);

<!-- more -->

# 缓冲区中的四个核心点

capacity (容量), limit (限制), position (位置), mark (标记), 大小关系为: 0 <= mark <= position <= limit <= capacity:

* capacity: 表示缓存最大存放的元素的数量, 缓存被创建后, 这个值不可变
* limit: 第一个不可写入或者读取的位置, 例如一个 ByteBuffer 的 limit 是 5, 那么表示这个缓存的 0 - 4 个字节可以用来存储数据, 从第 5 个字节开始就不能被写入和读取
* position: 当前读取到的位置
* mark: 标记位

如: ByteBuffer byteBuffer = ByteBuffer.wrap(new byte[]{'1', '2', '3', '4', '5'}), 初始值: capacity = 5, limit = 5, position = 0, mark = -1, 且初始值为 1, 2, 3, 4, 5, capacity 和 limit 和传入的数组大小一样大

# Buffer 类中的的方法

先来看看 Buffer 中的方法, 如下图表所示:

<img src="Java-NIO学习缓存API的使用/20190304223814.png"/>

## capacity()

capacity() 方法返回容量的大小

```java
public void bufferTest1() {
    // 此方法创建的 Buffer 的 capacity = 5, limit = 5
    ByteBuffer byteBuffer = ByteBuffer.wrap(new byte[]{'1', '2', '3', '4', '5'});
    log.info("capacity = {}", byteBuffer.capacity());
}
```

输出结果如下:

```text
capacity = 5
```

## limit() 和 limit(int)

获取 limit 的大小和重新设置 limit 值, 注意由于 limit 的值是可以动态设置的, 如果新设置的 limit 的值小于当前 position 的值, 此时会将 position 的值设置为新的 limit 值

```java
public void bufferTest2() {
    CharBuffer charBuffer = CharBuffer.wrap(new char[]{'a', 'b', 'c', 'd'});
    // 可以获取到 charBuffer[3] 的值
    log.info("charBuffer[3] = {}", charBuffer.get(3));

    // 重新设置 limit 为 4, 读取 charBuffer[3] 出现异常
    charBuffer.limit(3);
    log.info("charBuffer[4] = {}", charBuffer.get(3));
}
```

输出结果如下:

```text
charBuffer[3] = d
charBuffer[4] = charBuffer.get(3) 出现异常啦
```

## position() 和 position(int)

position() 方法可以获取到下一个将要写入到 Buffer 中或者读取 Buffer 位置的索引值, 如果 position 已经最大了, 继续放入值则会报异常, position(int) 方法则用于设置下一个将要写入到 Buffer 中或者读取 Buffer 位置的索引值

```java
public void bufferTest3() {
        CharBuffer charBuffer = CharBuffer.wrap(new char[4]);

    // 此时 position 为 0, charBuffer 由于没有存储值, 获取的值全是空值
    log.info("position = {}", charBuffer.position());
    log.info("charBuffer1 = {}, {}, {}, {}", charBuffer.get(0), charBuffer.get(1), charBuffer.get(2), charBuffer.get(3));

    // charBuffer 的值变成了 1, 2, 3, 4
    charBuffer.put("1");
    charBuffer.put("2");
    charBuffer.put("3");
    charBuffer.put("4");
    log.info("charBuffer2 = {}, {}, {}, {}", charBuffer.get(0), charBuffer.get(1), charBuffer.get(2), charBuffer.get(3));

    // 再添加一个 A, 出现异常
    try {
        charBuffer.put("A");
    } catch (Exception e) {
        log.error("出现异常啦2 = {}", e);
    }

    // 设置 position 的值为 0, 在存入值 A, charBuffer 的值变成了 A, 2, 3, 4, 在存入值 A 后的 position 值为 1
    charBuffer.position(0);
    charBuffer.put("A");
    log.info("charBuffer3 = {}, {}, {}, {}", charBuffer.get(0), charBuffer.get(1), charBuffer.get(2), charBuffer.get(3));
    log.info("position = {}", charBuffer.position());
}
```

输出结果如下:

```text
position = 0
charBuffer1 = , , ,
charBuffer2 = 1, 2, 3, 4
出现异常啦2 = {}
charBuffer3 = A, 2, 3, 4
position = 1
```

## mark()

mark(int) 方法用于在缓冲区中设置标记, 常和 reset() 方法配合使用, 调用 reset() 后会将 mark 值存入 position 中

* 注意 mark 的值默认是 -1, 如果没有设置 mark 的值, 调用 reset() 方法会报错, 不能将 -1 值设置给 position
* 注意由于 position 和 limit 的值可以被动态调整, 如果已经设置了 mark 值, 在调整 limit 或者 position 值后如果小于 mark 值, mark 值将会被自定抛弃并重置为 -1
* 注意 mark 的值在 API 中被设计为不可能设置为大于 position, mark 被设计成像是探险时设置的路标, 目的是方便返回时找到回去的路, 你不可能在你没到达某个目的地前就能在这个目的地设置一个路标

```java
public void bufferTest4() {
    CharBuffer charBuffer = CharBuffer.wrap(new char[4]);
    charBuffer.put('1');
    charBuffer.put('2');

    // 设置标记, 将当前 position 的值给 mark
    log.info("position = {}", charBuffer.position());
    charBuffer.mark();

    charBuffer.put('3');
    charBuffer.put('4');

    // charBuffer 的值为 1, 2, 3, 4, 调用 reset(), 将 mark 的值给 position, 再新添加值 A
    log.info("charBuffer = {}, {}, {}, {}", charBuffer.get(0), charBuffer.get(1), charBuffer.get(2), charBuffer.get(3));

    // charBuffer 的值变成了 1, 2, A, 4
    charBuffer.reset();
    charBuffer.put("A");
    log.info("charBuffer = {}, {}, {}, {}", charBuffer.get(0), charBuffer.get(1), charBuffer.get(2), charBuffer.get(3));

    // 下列操作, charBuffer 的值变成了 1, 2, C, 4
    charBuffer.position(2);
    charBuffer.mark();
    charBuffer.put("C");
    log.info("charBuffer = {}, {}, {}, {}", charBuffer.get(0), charBuffer.get(1), charBuffer.get(2), charBuffer.get(3));
}
```

输出结果如下:

```text
position = 2
charBuffer = 1, 2, 3, 4
charBuffer = 1, 2, A, 4
charBuffer = 1, 2, C, 4
```

## remaining()

返回剩余可读或者可写的空间, 他的值等于 limit - position

## hasRemaining()

判断是否还有剩余可读或者可写的空间

## isReadOnly()

是否时只读的缓存区

## isDirect()

是否是使用的直接缓存区, 通常我们使用的 Buffer 类, 我们成为直接缓存区, 在 JVM 中还有有一个中间缓存, 缓存读数据的过程, 将硬盘中的的数据拷贝到中间缓存, 然后从中间缓存中拷贝数据, 写也是这样一个操作过程的, 如果有大量的读写, 这样会大大降低对数据的处理, 直接使用直接缓冲区, 跳过将数据拷贝到中间缓存区这一步, 可以大大提高对数据的处理能力, 这种技术称之为零拷贝 (Buffer 类中只有 ByteBuffer 可以使用直接缓存区)

```java
public void bufferTest5() {
    ByteBuffer byteBuffer = ByteBuffer.allocateDirect(100);
    byteBuffer.put((byte) 1);
    log.info("isDirect = {}", byteBuffer.isDirect());
}
```

输出结果如下:

```text
isDirect = true
```

## clear()

将缓冲区还原为初始状态, 这个初始状态是 position = 0, limit= capacity, mark= -1, 要注意这个方法只是将缓冲区还原为初始状态, 缓存里面的数据并不会被擦除

## flip()

调用该方法调用该方法会令 limit = position, position = 0, mark = -1, 常用于在写入数据到缓存后调用该方法从缓存中读取数据

```java
public void bufferTest6() {
    ByteBuffer byteBuffer = ByteBuffer.allocateDirect(100);
    byteBuffer.put((byte) 0);
    byteBuffer.put((byte) 1);
    byteBuffer.put((byte) 2);
    byteBuffer.flip();
    log.info("byteBuffer[0] = {}", byteBuffer.get());
    log.info("byteBuffer[1] = {}", byteBuffer.get());
    log.info("byteBuffer[2] = {}", byteBuffer.get());
}
```

输出结果如下:

```text
byteBuffer[0] = 0
byteBuffer[1] = 1
byteBuffer[2] = 2
```

## rewind()

重置缓冲区位置, position = 0, mark = -1, capacity 和 limit 不变, 常用于需要重新读取或者写入缓存区的场景

## put(), get() 和放入其他类型的数据到缓存中

put() 方法和 get() 方法分别用于往缓存中写入和获取数据, 并且会使 position 自增, 但是要注意的是, 如果写入和读取的是指定 position 的值, position 则不会变化, 例如: put(int index, byte b) 方法不会使 position 的值加 1

## putChar(), putDouble() 等

对于 ByteBuffer 类型的缓存, putChat(char value) 用于在缓存中将指定的字符按照字节的方式写入到缓存中, char 是两个字节, 所以 position 会加 2, 同理, 还有 putDouble(), putShort() 等方法也是一样的效果

## slice()

从当前缓存区创建一个新的缓存区, 该缓存区的大小和限制当前缓存区剩余可读写大小一致, capacity = limit = 当前缓存区剩余可读写大小, position = 0, mark = -1, 如果当前缓存区里面有内容, 那么新的缓存区的内容和当前缓存区的当前 position 后面的值 (包括当前 position 的值) 一致的, 这两个缓存区里面的数据更改都会相互影响, 但是 position, limit 等都是相互独立的

```java
public void bufferTest7() {
    // 创建一个大小为 100 的缓存区, 在写入 5 个数据后, 可写的大小应该为 95
    ByteBuffer byteBuffer = ByteBuffer.allocate(100);
    byteBuffer.put((byte) 0);
    byteBuffer.put((byte) 1);
    byteBuffer.put((byte) 2);
    byteBuffer.put((byte) 3);
    byteBuffer.put((byte) 4);

    //  当前可写入的剩余大小 = 95
    log.info("当前可写入的剩余大小 = {}", byteBuffer.remaining());

    // 重新设置 position 值为 2, 然后调用 slice 创建新的缓存区域,
    // 新的缓存区 position = 0, mark = -1, capacity = limit = 100 -2
    byteBuffer.position(2);
    ByteBuffer newByteBuffer = byteBuffer.slice();

    // 新的缓存区 capacity = 98, limit = 98, position = 0
    log.info("新的缓存区 capacity = {}, limit = {}, position = {}",
            newByteBuffer.capacity(), newByteBuffer.limit(), newByteBuffer.position());

    // 写入数据会相互影响
    byteBuffer.put(2, (byte) 9);
    log.info("byteBuffer[2] = {}", byteBuffer.get(2));
    log.info("newByteBuffer[0] = {}", newByteBuffer.get(0));

    newByteBuffer.put(0, (byte) 2);
    log.info("byteBuffer[2] = {}", byteBuffer.get(2));
    log.info("newByteBuffer[0] = {}", newByteBuffer.get(0));
}
```

输出结果如下:

```text
当前可写入的剩余大小 = 95
新的缓存区 capacity = 98, limit = 98, position = 0

byteBuffer[2] = 9
newByteBuffer[0] = 9

byteBuffer[2] = 2
newByteBuffer[0] = 2
```

## 使用缓存区创建其他类型的缓存区

使用当前缓存区创建其他类型的缓存区, 这两个缓存区 position 等是相互独立的, 但是内容共享, 其他方法如 asDoubleBuffer(), asLongBuffer() 等, 转换后新的缓存的状态和转换前缓存类型和转换后缓存类型相关, 如 ByteBuffer 转 IntBuffer, 转换后的 IntBuffer 的 position = 0, mark = -1, capacity = limit 为原缓存的四分之一, 因为 int 占 4 个字节

## order() 和 order(ByteOrder order)

order() 获取当前缓存使用的字节序的模式, 默认情况下所有缓存使用的都是大端模式, order(ByteOrder order) 修改缓存区的字节序模式, 内存中存放的是字节, 对缓存区而言, 放入单个字节, 大端小端没有有影响

```java
public void bufferTest8() {
    ByteBuffer byteBuffer = ByteBuffer.allocate(100);
    ByteOrder byteOrder = byteBuffer.order();
    log.info("当前使用的字节序模式 = {}", byteOrder.toString());

    // 使用一个 int 数据, 占 4 个字节
    int value = 1;
    byteBuffer.order(byteOrder);
    byteBuffer.putInt(value);

    // 大端模式, int 四个字节, 存入缓存中的的数据如下
    // [00000000, 00000000, 00000000, 00000001]
    // 因此 byteBuffer[0] = 0, byteBuffer[3] = 1
    log.info("BIG_ENDIAN byteBuffer[0] = {}", byteBuffer.get(0));
    log.info("BIG_ENDIAN byteBuffer[3] = {}", byteBuffer.get(3));

    // 小端模式, int 四个字节, 存入缓存中的的数据如下
    // [00000001, 00000000, 00000000, 00000000]
    // 因此 byteBuffer[0] = 1, byteBuffer[3] = 0
    byteBuffer.order(ByteOrder.LITTLE_ENDIAN);
    byteBuffer.position(0);
    byteBuffer.putInt(value);
    log.info("LITTLE_ENDIAN byteBuffer[0] = {}", byteBuffer.get(0));
    log.info("LITTLE_ENDIAN byteBuffer[3] = {}", byteBuffer.get(3));
}
```

输出结果如下:

```text
当前使用的字节序模式 = BIG_ENDIAN
BIG_ENDIAN byteBuffer[0] = 0
BIG_ENDIAN byteBuffer[3] = 1
LITTLE_ENDIAN byteBuffer[0] = 1
LITTLE_ENDIAN byteBuffer[3] = 0
```

## compact()

压缩缓存区, 将当前缓存中 position 到 limit 之间的数据复制到缓存区的开始处, 假设这之间有数据 N, limit - position >= N, 调用该方法后, position = N + 1, mark = -1, limit 和 capacity 不变

## duplicate()

创建一个共享当前缓存区域内容的新的缓存区, 两个缓存的 position 等信息都是独立的, 底层使用的是同一个 byte 数组, 因此更改人一个缓存的内容会影响另一个缓存中的内容, 和 slice 方法创建的缓存区是情况类似

# 参考资料

* 书籍 - NIO于Scoket编程技术指南
* [浅析Linux中的零拷贝技术](https://www.jianshu.com/p/fad3339e3448)
* [零拷贝底层实现原理](https://juejin.im/entry/59b740fdf265da06633d02cf)
* [Java NIO 之 Buffer order](https://blog.csdn.net/will_awoke/article/details/25803725)