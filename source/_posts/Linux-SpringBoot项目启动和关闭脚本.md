---
title: Linux-SpringBoot项目启动和关闭脚本
date: 2018-12-28 23:25:47
categories: Linux
---

# 停止 SpringBoot 应用程序脚本步骤

服务器上发布项目时, 需要停止项目然后启动, 为了方便停止项目和启动项目时不需要频繁的输入指令, 写了两个 sh 脚本

## jsp -lv

l: 列出运行的 Java 进程
v: JVM 启动参数

```text
23863 sun.tools.jps.Jps -Denv.class.path=.:/usr/local/java//lib/dt.jar:/usr/local/java//lib/tools.jar -Dapplication.home=/usr/local/jdk1.8.0_101 -Xms8m
7809 wesure-wim-web-1.0.0-SNAPSHOT.jar -Xms1024m -Xmx1024m -Xmn512m
30369 wesure-scrm-web-1.0.0-SNAPSHOT.jar -Xms1024m -Xmx1024m -Xmn512m
25060 wesure-wimboss-1.0.0-SNAPSHOT.jar -Xms1024m -Xmx1024m -Xmn512m
31337 wesure-wimvip-1.0.0-SNAPSHOT.jar -Xms1024m -Xmx1024m -Xmn512m
18012 wesure-wimcrm-1.0.0-SNAPSHOT.jar -Xms1024m -Xmx1024m -Xmn512m
```

<!-- more -->

## jps -lv | grep wesure-wim-web

```shell
7809 wesure-wim-web-1.0.0-SNAPSHOT.jar -Xms1024m -Xmx1024m -Xmn512m
```

## jps -lv | grep wesure-wim-web | awk '{print $1}'

```shell
7809
```

## jps -lv | wesure-wim-web-1.0.0-SNAPSHOT.jar | awk '{print $1}' | xargs kill -9

将找出的 PID 作为 kill 指令的参数传递, kill 掉这个找到的 PID 进程

## 知道端口号时的写法

```shell
lsof -i:8082 | awk '{print $2}' | xargs kill -9
```

# stop.sh

```shell
jps -l | grep wesure-scrm-task-statistic-1.0.0-SNAPSHOT.jar | awk '{print $1}' | xargs kill -9
```

# start.sh

```shell
nohup java -Xms1024m -Xmx1024m -Xmn512m -jar wesure-scrm-web-1.0.0-SNAPSHOT.jar --spring.config.location=config/application.properties >/dev/null 2>&1 &

# SpringBoot 项目会自动到同级 config 目录下搜寻 application.properties, 上面的指令可以简写
nohup java -jar wesure-scrm-task-statistic-1.0.0-SNAPSHOT.jar >/dev/null 2>&1 &
```
