---
title: Docker-搭建Redis服务
date: 2018-03-31 15:48:26
categories: Docker
---

Redis 是一个使用ANSI C编写的开源、支持网络、基于内存、可选持久性的键值对存储数据库 (摘自维基百科), 对于 Redis 的入门学习资料可以参考链接 : `http://www.runoob.com/redis/redis-tutorial.html`, 本篇笔记主要记录使用 Docker 使用官方的镜像来搭建一个单机版的 Redis 服务。

# 了解 Redis 在容器中支持的 TAG

直接访问连接`https://hub.docker.com/_/redis/`即可了解到支持的 TAG，这里就不贴图了，在写这篇笔记的时候，默认的 latest 版本是 4.0，所以我也选取 4.0 作为等会要拉取镜像的 TAG。

# 拉取 Redis 镜像

``` shell
# 拉取镜像
docker pull redis:4.0

4.0: Pulling from library/redis
b0568b191983: Pull complete
6637dc5b29fe: Pull complete
7b4314315f15: Pull complete
2fd86759b5ff: Pull complete
0f04862b5a3b: Pull complete
2db0056aa977: Pull complete
Digest: sha256:39462a246a50859791bc6310c006c40e25506d617da496fcf6a69ebe3a204c15
Status: Downloaded newer image for redis:4.0
```

<!-- more -->

``` shell
# 查看镜像
docker images

REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
redis               4.0                 c5355f8853e4        3 days ago          107 MB
tomcat              latest              d636936d0d85        2 weeks ago         558 MB
mysql               5.7                 5195076672a7        2 weeks ago         371 MB
```

# 启动 Redis 容器

## 启动 Redis

-d 表示在后台运行, 容器名称为 redis_01, 使用默认的 6379 端口启动, 使用 redis:4.0 镜像启动容器。

``` shell
# 建立并启动容器
docker run -d --name redis_01 -p 6379:6379 redis:4.0 redis-server

# 查看容器是否启动
docker ps
```

## 连接容器中的 redis-server

连接容器中的 redis-server 前需要知道容器的 IP, 使用 docker inspect [容器名或容器ID] 可以查看到容器的 IP (该指令包含了容器的其他信息)，如下:

``` shell
# 查看容器的相关信息
docker inspect redis_01

# 部分信息如下
"Networks": {
    "bridge": {
        "IPAMConfig": null,
        "Links": null,
        "Aliases": null,
        "NetworkID": "8ef145b4b1d808dc70dfd894d5c06f697b500f5ca757f1db712abb9504ac2440",
        "EndpointID": "15547c1c24c5002f1fa37ec055f467de69ec08b6b9455c8ad39b2b689222b9d3",
        "Gateway": "172.17.0.1",
        "IPAddress": "172.17.0.4",
        "IPPrefixLen": 16,
        "IPv6Gateway": "",
        "GlobalIPv6Address": "",
        "GlobalIPv6PrefixLen": 0,
        "MacAddress": "02:42:ac:11:00:04"
    }
}
```

``` shell
# 使用 grep 截取 IP 地址
docker inspect redis_01 | grep IPAddress

"SecondaryIPAddresses": null,
"IPAddress": "172.17.0.4",
        "IPAddress": "172.17.0.4",
```

获取到 IP 地址后, 可以通过 redis-cli 指令连接 redis

``` shell
# -it 使用命令行交互模式启动
docker run -it redis:4.0 redis-cli -h 172.17.0.4 -p 6379
```

# 创建 Redis 容器时指定配置文件

在上面创建 redis 服务都是使用默认的配置启动的 (这个配置文件 redis.conf 默认是没有的, 需要我们自己创建)，我们创建一个容器并启动的时候，如果需要指定使用自己的配置文件时，可以使用以下指令实现。

``` shell
sudo docker run --name redis -p 6379:6379 -v $PWD/redis.config:/etc/redis/redis.config -v $PWD/data:/data -d redis:4.0 redis-server /etc/redis/redis.config --appendonly yes
```

```shell
sudo docker run --name redis -p 6379:6379 -v /home/lupw/redis/redis.config:/etc/redis/redis.config -v /home/lupw/redis/data:/data -d redis:4.0 redis-server /etc/redis/redis.config --appendonly yes
```

* -v /home/lupw/redis/redis.config:/etc/redis/redis.config:/etc/redis/redis.config 宿主机器的 redis.conf 文件挂载到容器指定目录里面;
* redis-server /usr/local/etc/redis/redis.conf 使用配置文件启动 redis-server, 配置文件路径为第一步挂载的路径;
* 持久化数据需要 --appendonly yes, 数据保存在容器的 /data 目录下看到 db 文件;

配置文件如下，** 有一个注意的地方, 在 docker 的 -d 模式启动 redis 时，配置文件中需要设置不能能以守护进程的方式运行, 即 daemonize 的值为 no, redis 默认也是 no, port 默认使用 6379 即可, 如果有修改, docker 启动时映射的时候使用设置的这个值就好了 (-p 6379:conf里面设置的值)**

挂载配置文件和挂载数据持久化的路径, 之后有配置文件的修改修改本地配置文件后重新启动容器即可

# 设置外网可访问

上一步已经挂载配置文件和挂载数据持久化的路径, 设置外网可访问时需要修改配置文件, 只需要修改宿主机 /etc/redis/redis.config 路径下的配置问价然后然后重新启动 redis 容器即可生效, 设置外网可访问的配置如下

```ini
修改配置文件, 一共三个地方需要修改
1. bind 127.0.0.1 注释掉改为 # bind 127.0.0.1
2. protected-mode yes 改为 protected -mode no
3. # requirepass foobared 去掉注释改为 requirepass yourpassword
```

启动完成后就可以访问了, 注意外网访问的时候 IP 是用宿主机的 IP 而不是使用 Docker 分配给容器的 IP:

```shell
udo docker restart redis
```

接着在 windows 上使用 `redis-cli.exe -h 地址 -p 端口号 -a 密码` 即可连接 redis

# 进入 redis 容器并保存数据

首先使用 `sudo docker exec -it redis /bin/bash` 进入容器

然后 redis-cli 进入 redis 客户端, 并保存数据

```text
root@4282e323ac04:/data# redis-cli
127.0.0.1:6379> set username lupengwei
OK
```