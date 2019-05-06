---
title: Java-RocketMQ的安装
date: 2018-05-10 11:13:06
categories: Java
---

消息队列(Message Queue, 简称 MQ), 消息中间件作为实现分布式消息系统可拓展、可伸缩性的关键组件, 具有高吞吐量、高可用等等优点。RocketMQ 是阿里一款开源的分布式消息系统, 基于高可用分布式集群技术, 提供低延时的、高可靠的消息发布与订阅服务。这篇笔记简单记录了 RocketMQ 的在服务器上的安装和启动方法。

# 下载、编译和环境变量的配置

## 下载地址

`https://github.com/apache/rocketmq`

## 安装 JDK 和 Maven

Maven 需要安装 JDK 支持, JDK 的安装和环境变量的配置这里就不记录了, 主要看看 Maven 在 Linux 上的安装和环境变量的配置, 首先去官网下载 Maven, 地址是 `https://maven.apache.org/download.cgi`, 然后解压缩后开始配置环境变量, 注意在 Linux 上的配置文件可以写在用户目录的 .bash_profile 文件里面, 在 Mac 上也有这个文件, 但是由于我安装了 on my zsh 的缘故, 在 Mac 上的 .bash_profile 文件里面配置环境变量没有效果, 需要在 .zshrc 文件里面去配置才会生效。

<!-- more -->

```sh
# Java 环境变量的配置
export JAVA_HOME=/Library/Java/JavaVirtualMachines/jdk1.8.0_151.jdk/Contents/Home
export PATH=$PATH:$JAVA_HOME/bin
export CLASSPATH=.:$JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar

# Maven 环境变量的配置
export M2_HOME=/Users/lupengwei/Maven/apache-maven-3.5.0
export PATH=$PATH:$M2_HOME/bin
```

## 解压并编译源码

下载好源码后, 解压缩, 在解压缩的目录下有一个 install.sh 的文件, 执行它开始编译, 注意这一步需要已经安装好了 Maven 环境, 执行成功后会在target目录下生成一个alibaba-rocketmq目录, 仅仅这个目录对我们是有用的, 其他的文件如果不需要均可以删除

## 配置环境变量

为了方便, 配置一下 RocketMQ 的环境变量。

```sh
# RocketMQ bin目录的环境变量的配置
export ROCKETMQ_HOME=/Users/lupengwei/RocketMQ/alibaba-rocketmq
export PATH=$PATH:$ROCKETMQ_HOME/bin
```

# 启动和关闭 RcoketMQ

## 启动 RcoketMQ

启动 RcoketMQ 要先启动 mqnamesrv, 然后再启动 broket,  所以首先启动 mqnamesrv，注意的是如果不去调整 jvm 的参数，默认的会使用 4G 内存，如果内存足够，可以忽略这一点，如果内存不足，就需要自己根据情况调整 jvm 参数了，见 `http://heht.net/articles/2018/04/03/1522752731021.html`。

```sh
nohup sh mqnamesrv >nameser.log 2>&1 &
```

启动后, 查看 nameser.log 的日志, 如果看见如下启动日志, 说明成功启动了

```txt
Java HotSpot(TM) 64-Bit Server VM warning: ignoring option PermSize=128m; support was removed in 8.0
Java HotSpot(TM) 64-Bit Server VM warning: ignoring option MaxPermSize=320m; support was removed in 8.0
Java HotSpot(TM) 64-Bit Server VM warning: UseCMSCompactAtFullCollection is deprecated and will likely be removed in a future release.
The Name Server boot success. serializeType=JSON
```

然后启动 mqbroker

```sh
nohup sh mqbroker -n 192.168.1.15:9876 >broker.log 2>&1 &
```

启动后, 查看 broker.log 的日志, 如果看见如下启动日志, 说明成功启动了

```txt
The broker[lupengweideMacBook-Pro.local, 192.168.1.15:10911] boot success. serializeType=JSON and name server is 192.168.1.15:9876
```

一般云主机上有一个内网的地址，使用默认的方式启动 broker 很有可能自动选择的 IP 是内网的，导致客户端连接失败，可以使用配置文件来启动 broker

启动后，发现自动在 Linux 目录下生成了以下文件，如图所示，除了 app，jdk，maven 和 rocketmq 目录是我创建的外，其他的均是 RocketMQ 启动后生成的。原因是没有在 broker 的配置文件中设置这些路径。

```sh
# mqbroker 生成一个配置文件
sh mqbroker -m broker.properties
```

配置如下

```ini
namesrvAddr=外网IP(或者你的内网IP):9876
brokerIP1=外网IP
brokerName=hht
brokerClusterName=DefaultCluster
brokerId=0
autoCreateTopicEnable=true
autoCreateSubscriptionGroup=true
```

接着启动 borker

```sh
nohup sh mqbroker -n 119.23.238.132:9876 -c ./broker.properties >broker.log 2>&1 &
```

![IMAGE](消息队列-RocketMQ的安装/1525946201065.jpg)

## 关闭 RcoketMQ

关闭 RcoketMQ 和启动顺序刚好相反, 要先关闭 broker,然后再关闭 nameser

```sh
# 关闭 broker
sh mqshutdown broker

# 关闭 mqnameserver
sh mqshutdown namesrv
```

# 学习资料

`http://www.bilibili.com/video/av11074519/`
`http://heht.net/articles/2018/04/03/1522752731021.html`