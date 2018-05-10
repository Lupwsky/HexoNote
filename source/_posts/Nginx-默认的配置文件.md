---
title: Nginx-默认的配置文件
date: 2018-04-25 22:51:52
categories:  Nginx
---

复制一份初始默认的配置文件, 方便之后使用。

# Nginx 默认的配置文件

<!-- more -->

```ini
# 1.13.2 默认的配置文件如下

#user  nobody;

# work 子进程数, Nginx 分成一个主进程和若干个 work 子进程, 主进程用于管理子进程, 子进程用于处理 Web 请求, 可以修改, 但太大无益, 因为会争夺 CPU, 一般设置为 CPU 数 * 核数, 例如 1 CPU 4 核的服务器, 设置为 4 即可
worker_processes  1;

# 日志文件的路径
#error_log  logs/error.log;
#error_log  logs/error.log  notice;
#error_log  logs/error.log  info;

# 进程文件路径
#pid        logs/nginx.pid;

events {
    # 配置 Nginx 的链接的特性
    # 一个 work 子进程支持的最大连接数, 根据服务器负载能力更改
    worker_connections  1024;
}

# 配置 Http 服务器的主要段, 里面的一个 server 就是一个虚拟主机
http {
    include       mime.types;
    default_type  application/octet-stream;

    # 日志文件的配置, 分片等
    #log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
    #                  '$status $body_bytes_sent "$http_referer" '
    #                  '"$http_user_agent" "$http_x_forwarded_for"';
    #access_log  logs/access.log  main;

    sendfile        on;

    #tcp_nopush     on;

    #keepalive_timeout  0;
    keepalive_timeout  65;

    # 是否压缩功能
    #gzip  on;

    # 一个 server 就是一个虚拟主机
    server {
        # 监听的端口
        # listen       192.168.1.1:80 default server; 了解下 default server
        listen       80;

        # 主机, 多个使用空格分隔
        server_name localhost example.net

        # 字符编码
        #charset koi8-r;

        # 可为每个虚拟机配置日志文件
        #access_log  logs/host.access.log  main;

        location / {
            root   html;
            index  index.html index.htm;
        }

        # 404页面
        #error_page  404              /404.html;

        # redirect server error pages to the static page /50x.html

        # 其他错误页面
        error_page   500 502 503 504  /50x.html;
        location = /50x.html {
            root   html;
        }

        # proxy the PHP scripts to Apache listening on 127.0.0.1:80
        #
        #location ~ \.php$ {
        #    proxy_pass   http://127.0.0.1;
        #}

        # pass the PHP scripts to FastCGI server listening on 127.0.0.1:9000
        #
        #location ~ \.php$ {
        #    root           html;
        #    fastcgi_pass   127.0.0.1:9000;
        #    fastcgi_index  index.php;
        #    fastcgi_param  SCRIPT_FILENAME  /scripts$fastcgi_script_name;
        #    include        fastcgi_params;
        #}

        # deny access to .htaccess files, if Apache's document root
        # concurs with nginx's one
        #
        #location ~ /\.ht {
        #    deny  all;
        #}
    }

    # 第二个 server
    # another virtual host using mix of IP-, name-, and port-based configuration
    #
    #server {
    #    listen       8000;
    #    listen       somename:8080;
    #    server_name  somename  alias  another.alias;

    #    location / {
    #        root   html;
    #        index  index.html index.htm;
    #    }
    #}

    # SSL 配置
    # HTTPS server
    #
    #server {
    #    listen       443 ssl;
    #    server_name  localhost;

    #    ssl_certificate      cert.pem;
    #    ssl_certificate_key  cert.key;

    #    ssl_session_cache    shared:SSL:1m;
    #    ssl_session_timeout  5m;

    #    ssl_ciphers  HIGH:!aNULL:!MD5;
    #    ssl_prefer_server_ciphers  on;

    #    location / {
    #        root   html;
    #        index  index.html index.htm;
    #    }
    #}

}
```
