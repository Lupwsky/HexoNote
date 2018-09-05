---
title: SpringBoot-配置logback打印日志
date: 2018-04-08 14:30:26
categories:  Spring
---

# Logback简介

Logback 是由 log4j 创始人设计的又一个开源日志组件, Logback 当前分成三个模块 : logback-core, logback- classic 和 logback-access。logback-core 是其它两个模块的基础模块, logback-classic 完整实现 SLF4J API 使你可以很方便地更换成其它日志系统如 log4j, log4j2 等其他日志框架, logback-access 访问模块与 Servlet 容器集成提供通过 Http 来访问日志的功能, 详情可以去[LogBack 官方网站](http://logback.qos.ch)了解。

# LogBack的使用

## 添加依赖

```xml
<dependency>  
    <groupId>ch.qos.logback</groupId>  
    <artifactId>logback-classic</artifactId>  
    <version>${logback.last.version}</version>  
</dependency>
```

<!-- more -->

## 配置文件

LogBack 加载配置文件的默认配置如下:

* 尝试在 classpath 下查找文件 logback-test.xml;
* 如果文件不存在, 则查找文件 logback.xml;
* 如果两个文件都不存在, logback 用 BasicConfigurator 自动对自己进行配置, 这会导致记录输出到控制台。

```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration debug="false">

    <!--定义日志文件的存储地址 勿在 LogBack 的配置中使用相对路径-->
    <property name="LOG_HOME" value="logs" />
    <property name="APP_NAME" value="grpc_server" />

    <!-- 控制台输出 -->
    <appender name="STDOUT" class="ch.qos.logback.core.ConsoleAppender">
        <encoder class="ch.qos.logback.classic.encoder.PatternLayoutEncoder">
            <!-- 格式化输出, %d : 日期; %thread : 线程名; %-5level : 级别, 从左显示5个字符宽度; %msg : 日志消息; %n : 换行符 -->
            <pattern>%d{yyyy-MM-dd HH:mm:ss} [%thread] %-5level %logger{50} - %msg%n</pattern>
        </encoder>
    </appender>

    <!-- 按照每天生成日志文件 -->
    <appender name="FILE"  class="ch.qos.logback.core.rolling.RollingFileAppender">
        <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
            <!-- 日志文件输出的文件名 -->
            <FileNamePattern>${LOG_HOME}/${APP_NAME}.log.%d{yyyy-MM-dd}.log</FileNamePattern>
            <!-- 日志文件保留天数 -->
            <MaxHistory>30</MaxHistory>
        </rollingPolicy>

        <!-- 日志文件最大的大小 -->
        <triggeringPolicy class="ch.qos.logback.core.rolling.SizeBasedTriggeringPolicy">
            <MaxFileSize>10MB</MaxFileSize>
        </triggeringPolicy>

        <encoder class="ch.qos.logback.classic.encoder.PatternLayoutEncoder">
            <pattern>%d{yyyy-MM-dd HH:mm:ss} [%thread] %-5level %logger{50} - %msg%n</pattern>
        </encoder>
    </appender>

    <!-- MyBatis 日志配置 -->
    <logger name="com.apache.ibatis" level="TRACE"/>
    <logger name="java.sql.Connection" level="DEBUG"/>
    <logger name="java.sql.Statement" level="DEBUG"/>
    <logger name="java.sql.PreparedStatement" level="DEBUG"/>

    <!-- 日志输出级别 -->
    <root level="INFO">
        <appender-ref ref="STDOUT" />
        <appender-ref ref="FILE" />
    </root>
</configuration>
```

日志文件的最大值的大小和日志文件的保留天数可以直接在一起配置，如下：

```xml
<appender name="FILE"  class="ch.qos.logback.core.rolling.RollingFileAppender">
    <rollingPolicy class="ch.qos.logback.core.rolling.SizeAndTimeBasedRollingPolicy">
            <fileNamePattern>${LOG_HOME}/${APP_NAME}.log.%d{yyyy-MM-dd}.log</fileNamePattern>
            <maxFileSize>10MB</maxFileSize>
            <maxHistory>30</maxHistory>
    </rollingPolicy>

    <encoder class="ch.qos.logback.classic.encoder.PatternLayoutEncoder">
        <pattern>%d{yyyy-MM-dd HH:mm:ss} [%thread] %-5level %logger{50} - %msg%n</pattern>
    </encoder>
</appender>
```

## 代码中使用

```java
public class Application {
    private final static Logger logger = LoggerFactory.getLogger(Application.class);

    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
        logger.warn("测试输出-------------");
    }
}
```

# SpringBoot项目中使用LogBack

SpringBoot 提供了一套日志框架支持系统，默认集成和使用的是 LogBack 日志框架，因此在 SpringBoot 项目中不必去单独的引入依赖了，直接添加配置文件即可使用, SpringBoot 加载配置文件官方推荐使用带 logback-spring.xml 文件作为配置的文件, 配置可以自动被读取, 当然也可以自定义文件名, 这个时候就需要在 application.properties 配置文件中去指定配置文件的路径了, 如下例子所示。

```ini
# logging
logging.config=classpath:logback-dev.xml
```

除了不需要单独添加依赖和注意 SpringBoot 默认的加载配置的文件名字外，其他的使用方法都是一样了。

关于时区的问题： `https://segmentfault.com/q/1010000010791397`