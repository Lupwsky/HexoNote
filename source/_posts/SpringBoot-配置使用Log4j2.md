---
title: SpringBoot-配置使用Log4j2
date: 2018-05-08 09:33:58
categories: Spring
---

# 添加相关依赖

在 pom.xml 中去除 spring-boot 默认的 logback 配置，引入 Log4j2 依赖 (spring-boot 在 1.4 版本后就不支持 log4j)。

```xml
<!-- spring boot web -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
    <exclusions>
        <exclusion>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-logging</artifactId>
        </exclusion>
    </exclusions>
</dependency>

<!-- Log4j2 -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-log4j2</artifactId>
</dependency>
```

<!-- more -->

# 创建Log4j2的配置文件

在classpath目录下创建log4j2.xml的配置文件，下面的配置文件是我再项目中使用的一个配置文件，按照每日日期和不同的日志级别切割日志。

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!--日志级别以及优先级排序: OFF > FATAL > ERROR > WARN > INFO > DEBUG > TRACE > ALL --><!--Configuration后面的status，这个用于设置log4j2自身内部的信息输出，可以不设置，当设置成trace时，你会看到log4j2内部各种详细输出--><!--monitorInterval：Log4j能够自动检测修改配置 文件和重新配置本身，设置间隔秒数-->
<configuration status="WARN" monitorInterval="30">
    <properties>
        <property name="log_home">/Users/lupengwei/home/files</property>
        <property name="file_name_info">upload_info</property>
        <property name="file_name_warn">upload_warn</property>
        <property name="file_name_error">upload_error</property>
    </properties>

    <appenders>
        <!--这个输出控制台的配置-->
        <console name="mConsole" target="SYSTEM_OUT">
            <PatternLayout pattern="[%d{HH:mm:ss:SSS}] [%p] - %l - %m%n"/>
        </console>

        <!--文件会打印出所有信息，这个log每次运行程序会自动清空，由append属性决定，这个也挺有用的，适合临时测试用-->
        <File name="log" fileName="${log_home}/current.log" append="false">
            <PatternLayout pattern="%d{HH:mm:ss.SSS} %-5level %class{36} %L %M - %msg%xEx%n"/>
        </File>
        <!-- 这个会打印出所有的info及以下级别的信息，每次大小超过size，则这size大小的日志会自动存入按年份-月份建立的文件夹下面并进行压缩，作为存档-->
        <RollingFile name="RollingFileInfo" fileName="${log_home}/${file_name_info}.log" filePattern="${log_home}/$${date:yyyy-MM}/${file_name_info}-%d{yyyy-MM-dd HH}-%i.log">
            <!--控制台只输出level及以上级别的信息（onMatch），其他的直接拒绝（onMismatch）-->
            <ThresholdFilter level="info" onMatch="ACCEPT" onMismatch="DENY"/>
            <PatternLayout pattern="[%d{HH:mm:ss:SSS}] [%p] - %l - %m%n"/>
            <Policies>
                <TimeBasedTriggeringPolicy/>
                <SizeBasedTriggeringPolicy size="100 MB"/>
            </Policies>
            <!-- DefaultRolloverStrategy属性如不设置，则默认为最多同一文件夹下7个文件，这里设置了20 -->
            <DefaultRolloverStrategy max="20"/>
        </RollingFile>

        <RollingFile name="RollingFileWarn" fileName="${log_home}/${file_name_warn}.log" filePattern="${log_home}/$${date:yyyy-MM}/${file_name_warn}-%d{yyyy-MM-dd HH}-%i.log">
            <ThresholdFilter level="warn" onMatch="ACCEPT" onMismatch="DENY"/>
            <PatternLayout pattern="[%d{HH:mm:ss:SSS}] [%p] - %l - %m%n"/>
            <Policies>
                <TimeBasedTriggeringPolicy/>
                <SizeBasedTriggeringPolicy size="100 MB"/>
            </Policies>
            <DefaultRolloverStrategy max="20"/>
        </RollingFile>

        <RollingFile name="RollingFileError" fileName="${log_home}/${file_name_error}.log" filePattern="${log_home}/$${date:yyyy-MM}/${file_name_error}-%d{yyyy-MM-dd HH}-%i.log">
            <ThresholdFilter level="error" onMatch="ACCEPT" onMismatch="DENY"/>
            <PatternLayout pattern="[%d{HH:mm:ss:SSS}] [%p] - %l - %m%n"/>
            <Policies>
                <TimeBasedTriggeringPolicy/>
                <SizeBasedTriggeringPolicy size="100 MB"/>
            </Policies>
            <DefaultRolloverStrategy max="20"/>
        </RollingFile>
    </appenders>

    <!--然后定义logger，只有定义了logger并引入的appender，appender才会生效-->
    <loggers>
        <!--过滤掉spring和mybatis的一些无用的DEBUG信息-->
        <logger name="org.springframework" level="info"/>
        <logger name="org.mybatis" level="info"/>

        <root level="debug">
            <appender-ref ref="mConsole"/>
            <appender-ref ref="RollingFileInfo"/>
            <appender-ref ref="RollingFileWarn"/>
            <appender-ref ref="RollingFileError"/>
        </root>
    </loggers>
</configuration>
```

# 指定使用log4j2的配置文件

application.properties文件中指定使用log4j2的配置文件。

```sh
# 指定log4j2的配置文件
logging.config=classpath:log4j2.xml
```

# 使用方法

可以使用 Lombok 的注解使用，在类的上面添加如下注解，然后使用 log.info 等输出不同的日志级别。

```java
@Log4j2(topic = "RollingFileInfo")
```

也可以使用 getLogger方法获取 log 对象，然后使用 log.info 等输出不同的日志级别。

```java
Logger log = LogManager.getLogger("RollingFileInfo");
```