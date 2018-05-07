---
title: Nginx-基础知识
date: 2018-04-25 21:30:29
categories:  Nginx
---

# Nginx 和其他 Web 服务器

1、 Ngnix 本来是用于做 Web 服务器，但是更多的是用于做负载均衡
2、 Apache 稳定，模块多，但是并发量相比 Nginx 低一些
3、 Tomcat(Java) 并发能力弱
4、 Web服务器排名：`http://news.netcraft.com`

# Nginx 的功能

1、 web 服务器
2、 反向代理
3、 邮件代理服务器
4、 负载均衡

<!-- more -->

# 选择 Nginx 的理由

1、 高并发连接，官方测试支持 5W 并发量(1s)，实际上2至4万的并发量，Nginx 采用的是 epoll 和 kqueque 网络 I/O 模型，Apache 使用的是传统的 select 模型
2、 内存消耗小，Nginx + PHP 在 3 万高并发情况下开启 10 个 Ngnix 进程仅消耗 150M 内存，开启 64 个 php-cli 进程消耗 128M 内存
3、 支持负载均衡
4、 支持反向代理
5、 相对硬件负载均衡的交换机动辄十几万、几十万，成本低廉，且可用于商用

注：压力测试工具 webbench

# Docker 启动一个 Nginx

```shell
docker run -p 80:80 --name mynginx -v $PWD/www:/www -v $PWD/conf/nginx.conf:/etc/nginx/nginx.conf -v $PWD/logs:/wwwlogs  -d nginx
docker exec -it mynginx /bin/bash
```

参数说明：
(1) `-p 80:80` [将容器的80端口映射到主机的80端口]
(2) `--name mynginx` [将容器命名为mynginx]
(3) `-v $PWD/www:/www` [将主机中当前目录下的www挂载到容器的/www]
(4) `-v $PWD/conf/nginx.conf:/etc/nginx/nginx.conf` [将主机中当前目录下的nginx.conf挂载到容器的/etc/nginx/nginx.conf]
(5) `-v $PWD/logs:/wwwlogs` [将主机中当前目录下的logs挂载到容器的/wwwlogs]
(6) 访问 `http://localhost/index.html` 出现 Nginx 欢迎页面表示启动成功

# Windows 上安装 Nginx 和启动

1、直接下载压缩包解压启动即可
2、conf、log、html分别存放的是配置文件、日志和静态页面文件

# 使用信号量控制 Nginx

常见的信号量如下，使用方法为 kill -信号名 pid，Nginx 的进程号查询 ps aux|grep nginx，每次查询进程号比较繁琐，可以从 pid 文件里面读取，`kill -信号量名 cat/xxx/path/log/ngnix.pid`。
(1) TERM，INI 直接关闭 Nginx 主进程
(2) QUIT 等所有 work 进程的请求结束后再关闭主进程
(3) HUP 开启新的子进程，并读取新的配置文件，再关闭旧的配置文件
(4) USR1 重读日志，在日志备份(手动)时使用，`mv error.log error.log.back`，`kill -USR1 pid` 才能将日志写入到 error.log 里面，否则会一直写到 error.log.back
(5) USR2 平滑的升级时使用
(6) WINCH 进程的请求结束后再关闭旧进程，配合 USR2 升级
