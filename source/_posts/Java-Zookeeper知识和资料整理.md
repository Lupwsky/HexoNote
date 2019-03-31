---
title: Java-Zookeeper知识和资料整理
date: 2019-02-23 10:36:08
categories: Java
---

关于 Zookeeper 的相关知识和学习资料的整理, 方便日后查阅

# Zookeeper 是什么

Zookeeper 是一个开源的分布式协调服务, 应用程序可以基于 Zookeeper 实现 `数据发布/订阅`, `负载均衡`, `命名服务`, `服务注册和发现`, `集群管理`, `选举`, `分布式锁` 和 `分布式队列` 等功能

# Zookeeper 的数据结构

Zookeeper 的数据结构是一个逻辑上的树形层次结构, 和 Linux 的文件系统类似, 可以理解为带有数据的目录结构, 在 Zookeeper 种称这种层次结构为 Znode 节点, 每一个层次结构的路径都对应一个 Znode, 最底层的 Znode 路径为 `/`, 由于 Zookeeper 并不是被设计为存储数据的, 因此 Znode 的数据最大只能存储 1M 的数据量

<!-- more -->

# Znode 节点的属性

使用 zkCli.sh 连接到客户端后使用 get 指令, 可以返回节点的各项属性和值, 使用 `zkCli.sh -timeout 5000 -server 127.0.0.1:2181` 连接到服务端后使用 get 指令获取节点属性值, 如下所示:

```text
[zk: localhost:2181(CONNECTED) 0] get /
cZxid = 0x0
mZxid = 0x0
pZxid = 0x2
ctime = Thu Jan 01 08:00:00 CST 1970
mtime = Thu Jan 01 08:00:00 CST 1970
cversion = 0
dataVersion = 0
aclVersion = 0
ephemeralOwner = 0x0
dataLength = 0
numChildren = 2
```

| 属性 | 含义 |
| --- | --- |
| cZxid | Znode 创建时的事务 ID, 这个 ID 的大小可以确定 Znode 的创建顺序 |
| mZxid | Znode 节点最后一次修改的事务 ID, 可以确定更新操作的顺序 |
| pZxid | Znode 的子节点最后一次更新时的事务 ID, 要注意的是这个更新指的是 Znode 的子节点列表发生变化而不是子节点的内容发生变化 |
| ctime | Znode 的创建时间 |
| mtime | Znode 的最后一次更新时间 |
| cversion | Znode 的子节点版本值, Znode 的子节点有更新时该值会增加 1, 和 pZxid 一样是指子节点列表发生变化而不是子节点的内容发生变化 |
| dataVersion | Znode 的数据版本值, 节点的数据每进行一次 set 操作, 该值就会增加 1 |
| aclVersion | Znode 的ACL (访问控制) 版本值 |
| numChildren | Znode 的子节点数量 |
| dataLength | Znode 节点中数据的长度 |
| ephemeralOwner | 0 表示这个节点是永久节点, 否则存放的是临时节点的会话 ID |

# Znode 节点的模式

Znode 的有以下模式, 且节点的类型模式在创建的时候就被创建了, 会一直存在, 不能被随时更改:

| 模式 | 说明 |
| --- | --- |
| PERSISTENT | 持久型模式, 默认的创建模式, 节点会一直存在, 除非主动删除节点 |
| EPHEMERAL | 临时型模式, 依赖与客户端会话, 会话结束, 临时节点即被删除, 利用临时节点可以方便的实现分布式锁, 服务发现注册和发现等功能 |
| PERSISTENT_SEQUENTIAL | 带序号的持久性模式, 在生成的子节点后面自动添加序号 (序号最大值为整型的最大值), 可以避免在同一个命名空间下的子节点不会有相同的路径 |
| EPHEMERAL_SEQUENTIAL | 带序号的临时型模式 |

注意: `Zookeeper 的临时节点不能拥有子节点`

参考资料:

* [ZooKeeper 的 Znode 剖析](https://blog.csdn.net/lihao21/article/details/51810395) - 节点类型和属性含义

# Znode 节点的操作

`节点的写操作是原子操作`, 关于节点的操作参考资料:

* [Zookeeper 客户端 zkCli 命令详解](https://blog.csdn.net/feixiang2039/article/details/79810102) - 包含节点的增删改查操作
* [分布式服务管理框架 Zookeeper 客户端 zkCli.sh 使用详解](https://blog.csdn.net/xyang81/article/details/53053642) - 该文主要讲解了对节点的操作, 提到了创建节点时的权限

## 创建节点

创建节点的指令: `create path [-s] [-e] data acl`

* -s: 带序号节点
* -e: 临时节点
* acl: 节点权限, c = 创建子节点权限, d = 删除子节点权限, r = 读取子节点列表的权限, w = 写权限, a = 管理子节点权限 

我这里主要只是记录下有序节点自动添加的序号的情况, 使用指令 `create -s /mdata` 执行三次, 然后使用 `ls /` 列出生成的 Znode 节点如下:

```text
[dubbo, zookeeper, mdata0000000001, mdata0000000002, mdata0000000003]
```

## 获取节点信息和数据

获取节点信息指令: `get path [watch]`

这里主要要了解的是 Zookeeper 的 watch 机制, 在获取节点信息或者节点数据的时候可以添加一个一次性的监听器, 当节点发生变化或者数据发生变化的时候就会通知客户端 (watcher)

参考资料:

* [ZooKeeper Watcher 注意事项](https://blog.csdn.net/oDaiLiDong/article/details/46473695)
* [Zookeeper 之 Watcher 监听事件丢失分析](https://blog.csdn.net/wo541075754/article/details/70207722)

# 其他参考资料

* [ZooKeeper 实战](https://segmentfault.com/a/1190000012185452)
* [可能是全网把 ZooKeeper 概念讲的最清楚的一篇文章](https://segmentfault.com/a/1190000016349824)
* [Zookeeper 客户端 Curator 使用详解](http://throwable.coding.me/2018/12/16/zookeeper-curator-usage)
* [阿里中间件团队博客 Zookeeper FAQ](http://jm.taobao.org/2013/10/07/zookeeper-faq/)
