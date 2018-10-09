---
title: Java-ETCD的安装和基础使用
date: 2018-09-05 23:13:21
categories:  Java
---

# 下载和安装ETCD

## 源码安装方式

### 下载和安装GO

[GO 官方网站](https://golang.org/) 下载并解压, 官网需要翻墙

### 配置环境变量

```shell
export GOROOT=/home/lupw/Golang/go
export PATH=$PATH:$GOROOT/bin
```

执行 go version 能查看版本信息即安装成功

<!-- more -->

### 下载ETCD源码并编译

```shell
go get -u -v https://github.com/coreos/etcd
./build
```

## 二进制文件方式安装

下载地址 [https://github.com/coreos/etcd/releases](https://github.com/coreos/etcd/releases/), 找到对应的系统版本下载安装即可

# 使用方法

为了方便使用, 创建了环境变量如下:

```shell
# ETCD 环境变量, 指定使用 API 的版本为 3, 不指定时默认使用的版本是 2
export ETCDCTL_API=3
export ETCD_HOME=/home/lupw/Etcd/etcd-v3.3.9-linux-amd64
export PATH=$PATH:$ETCD_HOME
```

## 单机启动

如果不更改端口号, 直接启动即可

```text
# 启动 etcd
nohup etcd
--name etcd0
--listen-peer-urls http://127.0.0.1:23790
--initial-advertise-peer-urls http://192.168.80.130:23790
--listen-client-urls http://127.0.0.1:2379
--advertise-client-urls http://192.168.80.130:2379
--initial-cluster etcd0=http://192.168.80.130:23790
--initial-cluster-token cluster0
--data-dir /home/lupw/Etcd/etcd-v3.3.9-linux-amd64
>etcd0.log 2>&1 &

# 启动参数说明
--name etcd0 本成员的名字
--listen-peer-urls http://127.0.0.1:23790 用于监听来自其他成员的交互信息
--initial-advertise-peer-urls http://192.168.80.130:23790 其他的成员通过这个地址和这个成员来交互信息, 需要保证其他成员能连接到这个地址 (一般使用公网 IP), 这个 IP 必须在集群描述信息中存在

--listen-client-urls http://127.0.0.1:2379 用于监听来自客户端的交互信息
--advertise-client-urls http://192.168.80.130:2379 客户端使用这个地址和集群中的这个成员来交互信息, 需要保证客户端能连接到这个地址 (一般使用公网 IP)

--initial-cluster etcd0=http://192.168.80.130:23790 集群的描述信息, 使用 [成员名称]=[地址] 的格式, 多个以逗号分隔, 逗号之间不能有空格, 这个成员会根据这里描述的地址去和其他成员交互信息
--initial-cluster-token cluster0 用于区分不同的集群, 一台服务器上又多个集群的话需要设置此值, 且值不能相同
--data-dir /home/lupw/Etcd/etcd-v3.3.9-linux-amd64 节点数据的存储目录

# 存放值和获取值
[lupw@localhost ~]$ etcdctl --endpoints=192.168.80.130:23790 put username lupengwei
OK
[lupw@localhost ~]$ etcdctl --endpoints=192.168.80.130:23790 get username
uername
lupengwei
```

## 单机多成员启动

### 启动

在同一台服务器上使用不同的端口号启动多个成员来模拟多成员集群

```text
nohup etcd --name etcd0 --listen-peer-urls http://192.168.80.130:23790 --initial-advertise-peer-urls http://192.168.80.130:23790 --listen-client-urls http://192.168.80.130:2379 --advertise-client-urls http://192.168.80.130:2379 --initial-cluster etcd0=http://192.168.80.130:23790,etcd1=http://192.168.80.130:23800,etcd2=http://192.168.80.130:23810 --initial-cluster-token cluster0 --data-dir /home/lupw/Etcd/etcd-v3.3.9-linux-amd64/etcd0 >etcd0.log 2>&1 &

nohup etcd --name etcd1 --listen-peer-urls http://192.168.80.130:23800 --initial-advertise-peer-urls http://192.168.80.130:23800 --listen-client-urls http://192.168.80.130:2380 --advertise-client-urls http://192.168.80.130:2380 --initial-cluster etcd0=http://192.168.80.130:23790,etcd1=http://192.168.80.130:23800,etcd2=http://192.168.80.130:23810 --initial-cluster-token cluster0 --data-dir /home/lupw/Etcd/etcd-v3.3.9-linux-amd64/etcd1 >etcd1.log 2>&1 &

nohup etcd --name etcd2 --listen-peer-urls http://192.168.80.130:23810 --initial-advertise-peer-urls http://192.168.80.130:23810 --listen-client-urls http://192.168.80.130:2381 --advertise-client-urls http://192.168.80.130:2381  --initial-cluster etcd0=http://192.168.80.130:23790,etcd1=http://192.168.80.130:23800,etcd2=http://192.168.80.130:23810 --initial-cluster-token cluster0 --data-dir /home/lupw/Etcd/etcd-v3.3.9-linux-amd64/etcd2 >etcd2.log 2>&1 &
```

### 查看成员列表

```text
etcdctl --endpoints=192.168.80.130:2379,192.168.80.130:2380,192.168.80.130:2381 --write-out=table member list
+------------------+---------+-------+-----------------------------+----------------------------+
|        ID        | STATUS  | NAME  |         PEER ADDRS          |        CLIENT ADDRS        |
+------------------+---------+-------+-----------------------------+----------------------------+
| 1a75caa138967ecd | started | etcd0 | http://192.168.80.130:23790 | http://192.168.80.130:2379 |
| 4d36558990e64509 | started | etcd2 | http://192.168.80.130:23810 | http://192.168.80.130:2381 |
| d84198c066efb916 | started | etcd1 | http://192.168.80.130:23800 | http://192.168.80.130:2380 |
+------------------+---------+-------+-----------------------------+----------------------------+
```

### 查看成员状态

```text
etcdctl --endpoints=192.168.80.130:2379,192.168.80.130:2380,192.168.80.130:2381 --write-out=table endpoint status
+---------------------+------------------+---------+---------+-----------+-----------+------------+
|      ENDPOINT       |        ID        | VERSION | DB SIZE | IS LEADER | RAFT TERM | RAFT INDEX |
+---------------------+------------------+---------+---------+-----------+-----------+------------+
| 192.168.80.130:2379 | 1a75caa138967ecd |  3.2.18 |   25 kB |     false |       142 |         17 |
| 192.168.80.130:2380 | d84198c066efb916 |  3.2.18 |   25 kB |      true |       142 |         17 |
| 192.168.80.130:2381 | 4d36558990e64509 |  3.2.18 |   25 kB |     false |       142 |         17 |
+---------------------+------------------+---------+---------+-----------+-----------+------------+
```

### 查看健康状态

```text
etcdctl --endpoints=192.168.80.130:2379,192.168.80.130:2380,192.168.80.130:2381 --write-out=table endpoint health
192.168.80.130:2380 is healthy: successfully committed proposal: took = 11.24872ms
192.168.80.130:2379 is healthy: successfully committed proposal: took = 11.060707ms
192.168.80.130:2381 is healthy: successfully committed proposal: took = 11.045883ms
```

### 在成员 etcd0 中存入值

```text
etcdctl --endpoints=192.168.80.130:23790 put username lupw
OK
```

### 在成员 etcd2 中取出值

```text
etcdctl --endpoints=192.168.80.130:23810 get username
username
lupw
```

# ETCD的Java客户端

现在开源的 Java 客户端有 jetcd 和 etcd4j, 官方的 jetcd 项目地址 `https://github.com/coreos/jetcd` 和 etcd4j 的项目地址 `https://github.com/jurmous/etcd4j`

## jetcd简单使用

### 添加依赖

```xml
<dependency>
  <groupId>com.coreos</groupId>
  <artifactId>jetcd-core</artifactId>
  <version>0.0.2</version>
</dependency>
```

### 使用

```java
Client client = Client.builder().endpoints("http://192.168.80.130:2379",
        "http://192.168.80.130:2380",
        "http://192.168.80.130:2381").build();
KV kvClient = client.getKVClient();
try {
    ByteSequence bsKey = ByteSequence.fromString("username");
    ByteSequence bsValue = ByteSequence.fromString("lupw");
    PutResponse putResponse = kvClient.put(bsKey, bsValue).get();
    log.warn("putResponse = {}", JSON.toJSONString(putResponse));

    GetResponse getResponse = kvClient.get(bsKey).get();
    log.warn("response = {}", JSON.toJSONString(getResponse));

    ByteSequence byteSequenceKey = getResponse.getKvs().get(0).getKey();
    ByteSequence byteSequenceValue = getResponse.getKvs().get(0).getValue();
    log.error(byteSequenceKey.toStringUtf8());
    log.error(byteSequenceValue.toStringUtf8());
} catch (InterruptedException | ExecutionException e) {
    e.printStackTrace();
}
```

## etcd4j简单使用

### 添加依赖

```xml
<dependency>
  <groupId>org.mousio</groupId>
  <artifactId>etcd4j</artifactId>
  <version>2.15.0</version>
</dependency>
```

# No help topic for 'put' 错误

需要 ETCDCTL_API=3 etcdctl put username lupw, 我上面是有配置环境变量使用 ETCDCTL_API=3 的, 但是忘记了使用 source .bash_profile 使配置环境变量生效

# 参考资料

* [Windows 下 GO 语言安装和环境变量的配置](https://blog.csdn.net/defonds/article/details/50538077)
* [Windows 下 GOPATH entry is relative 错误](https://blog.csdn.net/linuxshadow/article/details/49255737)* 
* [Linux 下 GO 语言安装和环境变量的配置](https://blog.csdn.net/u011054678/article/details/72877465)
* [ETCD 官方文档](https://coreos.com/etcd/docs/latest/getting-started-with-etcd.html)
* [ETCD 官方文档中文版](https://www.bookstack.cn/read/etcd/documentation-dev-guide-local_cluster.md)
* [搭建 ETCD 集群](https://segmentfault.com/a/1190000003852735)