---
title: Java-getProperties方法和getEnv方法
date: 2018-11-18 22:19:51
categories:  Java
---

# getProperties 方法

getProperties 方法可以获取系统相关属性, 比如: Java版本, 操作系统信息, 用户名等, 这个系统获取的相关属性的值可以在 JVM 启动的时候使用 `-D` 参数添加自定义的属性值, 也可以在项目中 (如 static 代码块中) 使用 System.setProperty 方法添加自定义的属性, 方便在其他的地方取用

```java
// Properties 类继承于 HashTable
Properties properties = System.getProperties();
properties.forEach((key, value) -> log.info("key = {}, value = {}", key, value));
```

<!-- more -->

这是我的电脑运行上面的代码输出的结果, 如下, 我整理成表格, 方便查看:

| key                           | 含义                                                                         | 输出示例                                                                          |
| :---------------------------- | :--------------------------------------------------------------------------- | :-------------------------------------------------------------------------------- |
| sun.cpu.isalist               | CPU 内核信息                                                                 | amd64                                                                             |
| sun.io.unicode.encoding       | 系统使用的 unicode 编码                                                      | UnicodeLittle                                                                     |
| sun.cpu.endian                | 系统使用的是大端模式或者小端模式                                             | little                                                                            |
| java.vendor.url.bug           | Java 运行时环境供应商 BUG 反馈地址                                           | `http://bugreport.sun.com/bugreport/`                                             |
| file.separator                | 文件分隔符 (在 UNIX 系统中是 "/")                                            | \                                                                                 |
| java.vendor                   | Java 运行时环境供应商                                                        | Oracle Corporation                                                                |
| sun.boot.class.path           | BootClassLoader 加载的类文件路径                                             | C:\Program Files\Java\jdk1.8.0_101\jre\lib\resources.jar; (其他更多)              |
| java.ext.dirs                 | 类加载的扩展目录路径                                                         | C:\Program Files\Java\jdk1.8.0_101\jre\lib\ext;C:\windows\Sun\Java\lib\ext        |
| java.version                  | Java 运行时环境版本                                                          | 1.8.0_101                                                                         |
| java.vm.info                  | JVM 的工作模式, `https://my.oschina.net/itblog/blog/507822`                  | mixed mode                                                                        |
| awt.toolkit                   |                                                                              | sun.awt.windows.WToolkit                                                          |
| user.language                 | 用户使用的语言                                                               | zh                                                                                |
| java.specification.vendor     | Java 运行时环境规范供应商                                                    | Oracle Corporation                                                                |
| sun.java.command              | 启动类的路径                                                                 | com.thread.excutor.SpringMain                                                     |
| java.home                     | Java 安装目录                                                                | C:\Program Files\Java\jdk1.8.0_101\jre                                            |
| sun.arch.data.model           | 系统的位数                                                                   | 64                                                                                |
| java.vm.specification.version | Java 虚拟机规范版本                                                          | 1.8                                                                               |
| java.class.path               | Java 类路径                                                                  | C:\Program Files\Java\jdk1.8.0_101\jre\lib\charsets.jar; (其他更多)               |
| user.name                     | 用户名                                                                       | lupw                                                                            |
| file.encoding                 | 文件使用的编码                                                               | UTF-8                                                                             |
| java.specification.version    | Java 运行时环境规范版本                                                      | 1.8                                                                               |
| java.awt.printerjob           | 控制打印的主要类                                                             | sun.awt.windows.WPrinterJob                                                       |
| user.timezone                 | 用户时间区域                                                                 | Asia/Shanghai                                                                     |
| user.home                     | 用户的主目录                                                                 | C:\Users\lupw                                                                   |
| os.version                    | 操作系统版本号                                                               | 10.0                                                                              |
| sun.management.compiler       |                                                                              | HotSpot 64-Bit Tiered Compilers                                                   |
| java.specification.name       |                                                                              | Java Platform API Specification                                                   |
| java.class.version            | Java 类格式版本号                                                            | 52.0                                                                              |
| java.library.path             | 加载库路径                                                                   | C:\Program Files\Java\jdk1.8.0_101\bin;C:\windows\system32;C:\windows; (其他更多) |
| sun.jnu.encoding              | 操作系统默认编码, `https://blog.csdn.net/miracle_8/article/details/80289893` | GBK                                                                               |
| os.name                       | 操作系统的名称                                                               | Windows 10                                                                        |
| user.variant                  |                                                                              |                                                                                   |
| java.vm.specification.vendor  | Java 虚拟机规范供应商                                                        | Oracle Corporation                                                                |
| java.io.tmpdir                | 默认的临时文件路径                                                           | C:\Users\lupw\AppData\Local\Temp\                                               |
| line.separator                | 行分隔符 (在 UNIX 系统中是 "/n")                                             | \n                                                                                |
| java.endorsed.dirs            | endorsed 加载的类路径                                                        | C:\Program Files\Java\jdk1.8.0_101\jre\lib\endorsed                               |
| os.arch                       | 操作系统的架构                                                               | amd64                                                                             |
| java.awt.graphicsenv          |                                                                              | sun.awt.Win32GraphicsEnvironment                                                  |
| java.runtime.version          | Java 运行环境的版本                                                          | 1.8.0_101-b13                                                                     |
| java.vm.specification.name    | Java 虚拟机规范名称                                                          | Java Virtual Machine Specification                                                |
| user.dir                      | 当前项目工作目录                                                             | D:\Work\hello-grpc                                                                |
| user.country                  | 用户所属国家                                                                 | CN                                                                                |
| user.script                   |                                                                              |                                                                                   |
| sun.java.launcher             | 加载模式                                                                     | SUN_STANDARD                                                                      |
| sun.os.patch.level            |                                                                              |                                                                                   |
| java.vm.name                  | 虚拟机名称                                                                   | Java HotSpot(TM) 64-Bit Server VM                                                 |
| file.encoding.pkg             |                                                                              | sun.io                                                                            |
| path.separator                | 路径分隔符 (在 UNIX 系统中是 ":")                                            | ;                                                                                 |
| java.vm.vendor                | 虚拟机供应商                                                                 | Oracle Corporation                                                                |
| java.vendor.url               | Java 运行时环境供应商官网                                                    | `http://java.oracle.com/`                                                         |
| sun.boot.library.path         | JRE 根路径                                                                   | C:\Program Files\Java\jdk1.8.0_101\jre\bin                                        |
| java.vm.version               | 虚拟机版本                                                                   | 25.101-b13                                                                        |
| java.runtime.name             | 虚拟机运行时环境名称                                                         | Java(TM) SE Runtime Environment                                                   |

# getEnv 方法

getEnv 方法可以获取设置的环境变量 (UNIX 系统的 .bashrc, .profile 等文件里面配置), 常见的如 Java 的 JAVA_HOME, CLASSPATH 的环境变量等

```java
Map<String, String> env = System.getenv();
env.forEach((key, value) -> log.info("key = {}, value = {}", key, value));
```

输出结果如下:

```text
// 这里贴出部分环境变量
key = USERDOMAIN_ROAMINGPROFILE, value = WESURE
key = LOCALAPPDATA, value = C:\Users\lupw\AppData\Local
key = PROCESSOR_LEVEL, value = 6
key = IntelliJ IDEA, value = D:\JetBrains\IntelliJ IDEA 183.3795.13\bin;
key = USERDOMAIN, value = WESURE
key = LOGONSERVER, value = \\SZ-ADDNS-162
key = JAVA_HOME, value = C:\Program Files\Java\jdk1.8.0_101 
```
