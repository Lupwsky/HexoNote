---
title: Java-JVM的常用的GC日志参数和日志解读
date: 2019-05-29 11:10:39
categories: Java
---

# -XX:PrintGC

输出简单的 GC 日志, 等同于使用 `-verbose:gc`, 当我们主动调用 System.gc() 或者 Runtime.getRuntime().gc() 触发 GC 时, 输出示例如下

```text
[GC (System.gc())  36680K->15974K(262144K), 0.0046979 secs]
[Full GC (System.gc())  15974K->12176K(262144K), 0.0595575 secs]
```

日志分析示例:

* `GC` 表示这次 GC 是 Minor GC, 发生在新生代
* `(System.gc())` 表示这次 GC 是调用 System.gc() 触发的, 这种方式触发的 GC 会进行 Full GC
* `36680K->15974K(262144K)` 表示堆区在 GC 前后的大小, 由 36680K 变为 15974K
* `(262144K)` 表示当前堆区的总大小为 262144K
* `0.0046979 secs` 表示此次 GC 耗费的时间

<!-- more -->

同时也可以了解到调用 System.gc() 方法不仅会触发 Minor GC, 还会触发 Full GC (Major GC), 建议在一般业务程序中不要显示的使用 System.gc() 去触发 GC

<div class="note default"><p> 和 -XX:PrintGCDetails 参数一起使用时, 优先使用 -XX:PrintGCDetails 参数, 且添加了 -XX:PrintGCDetails 参数, 相当于自动设置了添加了 -verbose:gc 和 -XX:PrintGC 参数 </p></div>

# -XX:PrintGCDetails

输出 GC 详细的日志, 包括 GC 使用的收集器, 发生 GC 的区域, GC 空间收集情况等信息, 主动调用 System.gc() 触发 GC 时, 输出示例如下

```text
[GC (System.gc()) 
    [PSYoungGen: 32012K->8324K(167936K)] 39505K->15825K(266240K), 0.0047314 secs] 
    [Times: user=0.00 sys=0.00, real=0.00 secs]

[Full GC (System.gc()) 
    [PSYoungGen: 8324K->0K(167936K)] 
    [ParOldGen: 7500K->12256K(98304K)] 15825K->12256K(266240K), [Metaspace: 32483K->32483K(1079296K)], 0.0622250 secs]
    [Times: user=0.36 sys=0.00, real=0.06 secs]
```

GC 日志分析示例:

* `PSYoungGen` 表示使用的垃圾收集器
* `32012K->8324K(167936K)` 年轻代堆区空间由 32012K 减少为 8324K, 总大小为 167936K
* `39505K->15825K(266240K)` 整个堆区由 39505K 减少为 15825K, 总大小为 266240K
* `0.0047314 secs` GC 表示此次 GC 耗费的时间
* `[Times: user=0.00 sys=0.00, real=0.00 secs]` 耗时详情, 分别是用户态耗时, 核心态耗时和实际耗时

Full GC 日志分析示例:

* `[PSYoungGen: 8324K->0K(167936K)]` 年轻代 GC 使用的收集器, 年轻代堆区变化
* `[ParOldGen: 7500K->12256K(98304K)] 15825K->12256K(266240K), [Metaspace: 32483K->32483K(1079296K)], 0.0622250 secs]` 老年代空间变化和总空间变化, 元数据区变化以及 GC 表示此次 GC 耗费的时间
* `[Times: user=0.36 sys=0.00, real=0.06 secs]` 耗时详情

<div class="note default"><p>不同收集器的 GC 日志也许会有些不同, 但是各项参数的含义都是一样的 </p></div>

# -XX:+PrintGCHeapAtGC

在发生 GC 时输出 GC 前后各区域空间详细的对比信息, 对 -XX:PrintGCDetails 更加详细的补充, 使用如下配置进行测试并打印 GC 日志

```text
-XX:+PrintGCDateStamps
-XX:+PrintGCDetails
-XX:+PrintHeapAtGC
```

输出的 GC 日志和对日志文件的分析示例就直接写在注释里面, 如下所示

```text
// GC 前的各区域内存空间使用情况
{Heap before GC invocations=7 (full 1):
    // 年轻代使用的收集器, 内存空间总大小和当前使用的空间大小
    PSYoungGen total 141824K, used 68472K [0x000000076b400000, 0x000000077b180000, 0x00000007c0000000)
        // eden 区空间使用情况
        eden space 131072K, 44% used      [0x000000076b400000,0x000000076ec63380,0x0000000773400000)
        // from 区的空间使用情况
        from space 10752K, 99% used       [0x0000000773400000,0x0000000773e7af00,0x0000000773e80000)
        // to 区的空间使用情况
        to   space 12800K, 0% used        [0x000000077a500000,0x000000077a500000,0x000000077b180000)

    // 老年代使用的收集器, 内存空间总大小和当前使用空间大小
    ParOldGen       total 82944K, used 7265K [0x00000006c1c00000, 0x00000006c6d00000, 0x000000076b400000)
        object space 82944K, 8% used         [0x00000006c1c00000,0x00000006c2318678,0x00000006c6d00000)
    
    // 元数据区域内存空间大小的使用情况
    Metaspace       used 32471K, capacity 34086K, committed 34304K, reserved 1079296K
        class space    used 4227K, capacity 4542K, committed 4608K, reserved 1048576K

// GC 日志
2019-05-29T16:17:44.684+0800: [GC (System.gc()) 
    [PSYoungGen: 68472K->9691K(246272K)] 75738K->16965K(329216K), 0.0052534 secs] 
    [Times: user=0.11 sys=0.02, real=0.00 secs] 

// GC 后的各区域内存空间使用情况
Heap after GC invocations=7 (full 1):
    PSYoungGen total 246272K, used 9691K [0x000000076b400000, 0x000000077b280000, 0x00000007c0000000)
        eden space 233472K, 0% used           [0x000000076b400000,0x000000076b400000,0x0000000779800000)
        from space 12800K, 75% used           [0x000000077a500000,0x000000077ae76f40,0x000000077b180000)
        to   space 13312K, 0% used            [0x0000000779800000,0x0000000779800000,0x000000077a500000)
 
    ParOldGen       total 82944K, used 7273K [0x00000006c1c00000, 0x00000006c6d00000, 0x000000076b400000)
        object space 82944K, 8% used         [0x00000006c1c00000,0x00000006c231a678,0x00000006c6d00000)
    
    Metaspace       used 32471K, capacity 34086K, committed 34304K, reserved 1079296K
    class space    used 4227K, capacity 4542K, committed 4608K, reserved 1048576K
}
```

类似 [0x000000076b400000, 0x000000077b180000, 0x00000007c0000000) 的这一串数据表示当前区域内存地址的下界, 当前下届和上届, 以十六进制表示 

# -XX:+PrintGCTimeStamps

在输出 GC 日志的时候输出时间戳, 这个时间戳是相对于虚拟机的启动时间, GC 日志示例如下

```text
41.022: [GC (System.gc()) 
    [PSYoungGen: 36378K->8548K(155136K)] 44029K->16207K(257536K), 0.0051510 secs]
    [Times: user=0.00 sys=0.00, real=0.00 secs] 
```

<div class="note default"><p> 和 -XX:PrintGCDateStamps 参数一起使用时, 优先使用 -XX:PrintGCDateStamps 参数 </p></div>

# -XX:+PrintGCDateStamps

在输出 GC 日志的时候输出 GC 发生的时间, GC 日志示例如下

```text
2019-05-29T15:11:21.700+0800: 4.555: [GC (System.gc()) 
    [PSYoungGen: 26799K->8122K(166400K)] 34557K->15887K(265728K), 0.0052315 secs]
    [Times: user=0.00 sys=0.00, real=0.00 secs] 
```

# -XX:+PrintReferenceGC

在 GC 的时候打印系统中的软引用, 弱引用和虚引用等引用的数量信息, GC 日志示例如下

```text
2019-05-29T16:42:01.149+0800: [GC (System.gc()) 
    2019-05-29T16:42:01.153+0800: [SoftReference, 0 refs, 0.0000393 secs]
    2019-05-29T16:42:01.153+0800: [WeakReference, 908 refs, 0.0000831 secs]
    2019-05-29T16:42:01.153+0800: [FinalReference, 359 refs, 0.0003339 secs]
    2019-05-29T16:42:01.153+0800: [PhantomReference, 0 refs, 1 refs, 0.0000077 secs]
    2019-05-29T16:42:01.153+0800: [JNI Weak Reference, 0.0002381 secs]
    [PSYoungGen: 29730K->8327K(161280K)] 37369K->15974K(261120K), 0.0050600 secs] 
    [Times: user=0.00 sys=0.00, real=0.01 secs] 

2019-05-29T16:42:01.154+0800: [Full GC (System.gc())
    2019-05-29T16:42:01.159+0800: [SoftReference, 0 refs, 0.0000464 secs]
    2019-05-29T16:42:01.159+0800: [WeakReference, 262 refs, 0.0000472 secs]
    2019-05-29T16:42:01.159+0800: [FinalReference, 30 refs, 0.0000134 secs]
    2019-05-29T16:42:01.159+0800: [PhantomReference, 0 refs, 2 refs, 0.0000267 secs]
    2019-05-29T16:42:01.159+0800: [JNI Weak Reference, 0.0002375 secs]
    [PSYoungGen: 8327K->0K(161280K)] 
    [ParOldGen: 7647K->12206K(99840K)] 15974K->12206K(261120K), 
    [Metaspace: 32469K->32469K(1079296K)], 0.0618857 secs]
    [Times: user=0.25 sys=0.00, real=0.06 secs] 
```

# -Xloggc

输出 GC 日志到文件, 方便排查问题, 如果不配置路径, GC 日志将直接在控制台输出,格式为 `-Xloggc:/paht/xxx.log`, 添加如下 JVM 参数

```text
-javaagent:D:/Work/hello-grpc/javaagentpackage/target/javaagentpackage-jar-with-dependencies.jar
-XX:+PrintGCDetails
-XX:+PrintGCDateStamps
-Xloggc:D:/Work/hello-grpc/logs/javaagent_gc.log
```

运行项目后, 打开 javaagent_gc.log 日志, 部分 GC 日志如下

```text
Java HotSpot(TM) 64-Bit Server VM (25.101-b13) for windows-amd64 JRE (1.8.0_101-b13),
    built on Jun 22 2016 01:21:29 by "java_re" with MS VC++ 10.0 (VS2010)
Memory: 4k page, physical 16658264k(8622712k free), swap 30289752k(15016868k free)
CommandLine flags: -XX:-BytecodeVerificationLocal 
    -XX:-BytecodeVerificationRemote 
    -XX:InitialHeapSize=266532224 
    -XX:+ManagementServer
    -XX:MaxHeapSize=4264515584 
    -XX:+PrintGC 
    -XX:+PrintGCDateStamps 
    -XX:+PrintGCDetails 
    -XX:+PrintGCTimeStamps 
    -XX:TieredStopAtLevel=1 
    -XX:+UseCompressedClassPointers 
    -XX:+UseCompressedOops 
    -XX:-UseLargePagesIndividualAllocation 
    -XX:+UseParallelGC 
2019-05-29T16:01:09.800+0800: 0.737: [GC (Allocation Failure) 
    [PSYoungGen: 65536K->6874K(76288K)] 65536K->6890K(251392K), 0.0061366 secs] 
    [Times: user=0.00 sys=0.00, real=0.01 secs] 
2019-05-29T16:01:10.094+0800: 1.031: 
    [GC (Allocation Failure) [PSYoungGen: 72410K->9026K(76288K)] 72426K->9050K(251392K), 0.0108208 secs] 
    [Times: user=0.00 sys=0.00, real=0.01 secs]
```

<div class="note default"><p> 注意里面其中有一行 CommandLine flags 开头的日志, 输出了 JVM 默认加载的参数 </p></div>

# -XX:+UseGCLogFileRotation

是否启用 GC 日志文件自动转储, GC 日志文件按大小切割时需要设置, 使用示例如下

```text
-XX:+UseGCLogFileRotation
```

# -XX:GCLogFileSize

控制 GC 日志文件的大小, 用于 GC 日志的切割, 不宜设置太大, 太大由于写日志需要进行 I/O 操作, I/O 操作需要在用户态和和心态之间切换, 会直接影响 user time 大小和 sys time 大小, 最终影响到 real time 大小, real time 就是 GC 耗费的时间, 使用示例如下

```text
// [Times: user=0.00 sys=0.00, real=0.01 secs] 
-XX:GCLogFileSize=50M
```

# -XX:NumberOfGClogFiles

GC 日志文件最多保存的个数, 使用示例如下

```text
-XX:NumberOfGClogFiles=10
```