---
title: Nginx-配置虚拟主机和日志
date: 2018-04-25 22:57:52
categories:  Nginx
---

虚拟主机的配置主要是配置配置文件中的 http 块中的 server，每一个 server 就是一个虚拟主机，可以配置多个。

# 虚拟主机的配置

```ini
# 一个 server 就是一个虚拟主机
server {
    # 监听的IP、端口
    listen       80;

    # 域名或者 IP，多个使用空格分隔
    server_name localhost example.net 192.168.1.177;

    # 字符编码
    charset UTF-8;

    # 可为每个虚拟机配置日志文件
    access_log  logs/host.access.log  main;

    # 映射到哪个目录文件下，root 路径可以是相对路径或者是绝对路径，相对路径是相对于于 Nginx 目录
    location / {
        root   html;
        index  index.html index.htm;
    }

    # 404页面
    error_page  404              /404.html;

    # 其他错误页面
    error_page  500 502 503 504  /50x.html;

    # 指定 /50x.html 映射的路径
    location = /50x.html {
        root   html;
    }
}
```

<!-- more -->

# 日志文件的配置

Nginx 针对不同的虚拟主机可以做不同的日志，日志文件主要是在虚拟机配置文件中的 access_log logs/host.access.log  main 段，这个配置段落包含了如下的内容：

1. 日志文件存放在 logs 目录下的 host.access.log 文件下；
2. 使用的格式是 main，格式可以使用自己定义的格式，main 格式的定义可以在默认的配置文件中可以看见。

``` ini
# main 日志格式的定义
log_format main '$remote_addr - $remote_user [$time_local] "$request" '
                '$status $body_bytes_sent "$http_referer" '
                '"$http_user_agent" "$http_x_forwarded_for"';

# main 格式完成的输出的日志例子
127.0.0.1 - - [13/Apr/2018:00:19:59 +0800] "GET /favicon.ico HTTP/1.1" 404 572 "http://localhost/imdex.html" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/65.0.3325.181 Safari/537.36"

# main 格式中字段的含义
$remote_addr = 127.0.0.1，远程访问的地址
$remote_user = ''
$time_local = 13/Apr/2018:00:19:59 +0800
$request = GET /favicon.ico HTTP/1.1，GET 方法请求，使用的 HTTP/1.1 协议
$status = 404，请求的状态，其让例如200等
$body_bytes_sent = 572，表示发送的字节大小
$http_referer = http://localhost/imdex.html，访问的上一个链接地址来自于哪
$http_user_agent = Mozilla/5.0 (Windows NT 10.0, Win64, x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/65.0.3325.181 Safari/537.36
$http_x_forwarded_for =
```

# 日志的切割

项目一直运行，并发量大的时候，日志文件就会越来越大，所以有必要对日志进行切割，可以使用 USR1 信号量 + Linux 的定时任务来实现。

先熟悉一下 dete 指令，生成日期时需要：

```sh
# date 帮助指令
date --help

# 查看昨天的日期
date -d yesterday

# 查看昨天的日期并格式化
date -d yesterday +%Y%m%d
```

创建 runlog.sh 的测试脚本，获取格式化后的日期字符串，之后用于拼接成分割后的日志文件名，内容如下：

```sh
#!/bin/bash

# 注意要带反引号，才会获取到日期的结果，否则会直接将 date -d yesterday +%Y%m%d 作为字符串输出，或着使用 $ 符号
# echo $(date -d yesterday +%Y%m%d)
echo `date -d yesterday +%Y%m%d`
```

保存脚本文件后运行，执行指令 sh runlog.sh 进行测试，上面的测试没有问题后，使用下面的 sh 并配合 Linux 的定时任务就可以完成按天分割日志，并按每月目录整理日志，关于定时任务指令 crontab 的文章参考 `http://www.cnblogs.com/peida/archive/2013/01/08/2850483.html`

```sh
#!/bin/bash
# 日志文件的路径
LOGPATH=/User/lupw/logs/access.log
# 日志文件的备份路径
BACKPATH=/user/lupw/logs/backlog/$(date -d yesterday +%Y%m)
# 按月创建目录，不存在的时候创建
mkdir -p $BACKPATH
# 备份日志文件的路径
BACK_FILE_PATH=$BACKPATH/$(date -d yesterday +%Y%m%d).access.log
# 移动原日志文件路径到备份文件的目录
mv $LOGPATH $BACK_FILE_PATH
# 建立新的日志文件
touch $LOGPATH
# 发信号重读日志
kill -USR1 `cat /User/lupw/logs/ngnix.pid`
```