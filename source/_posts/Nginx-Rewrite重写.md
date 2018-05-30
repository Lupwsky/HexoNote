---
title: Nginx-Rewrite重写
date: 2018-05-04 00:17:29
categories: Nginx
---

Rewrite 的可以重新定向 (会重写用户的请求地址，但是不会修改问号后面参数) URL 地址，通过在配置文件中重写 URL，可以让网站中的 URL 符合某种条件后跳转到指定的 URL，如重定向到 301，404 等页面，例如因为需求更改了项目的结构，用户之前收藏的链接再去访问就会出现错误，Rewrite 就可以用于解决这种问题。Rewrite 还可以完成防盗链的功能。源码的实现可以查看 ngx_http_rewrite_module 模块，官方地址 `https://nginx.org/en/docs/`。

# Rewrite 中的指令

Rewrite 的重写规则既可以放在 server，也可以放在 location 里面。

## if

if 指令的作用域为 server、location，在使用的时候要注意的是 if 的后面必须要有一个空格。

if 指令的的条件可以是：

| 操作符 | 作用 |
| --- | --- |
| 一个变量 | 变量不为空或者不为 0 的字符串则条件为真 |
| ==, != | 比较的一个变量和字符串 |
| ~, ~* | 与正则表达式匹配的变量，如果这个正则表达式中包含 "}"，则整个表达式需要用 " 或 ' |
| -f, !-f | 检查一个文件是否存在 |
| -d, !-d | 检查一个目录是否存在 |
| -e, !-e | 检查一个文件、目录是否存在 |
| -x, !-x | 检查一个文件是否可执行 |

使用实例：

```ini
if ($request_method == POST){
    return 404;
}
```

## return

return 指令的作用域是 server、location 和 if，return 指令可以中断请求，并且返回 一个 HTTP 状态码。

```ini
if ($request_method == POST){
    # 返回 404 错误码
    return 404;
}
```

## break

break 指令的作用域是 server、location 和 if，break 指令可以阻止后面的 rewrite 指令执行。

## set

set 指令的作用域是 server、location 和 if，set 指令可以初始化或设置一个变量。

```ini
location /test {
    set $foo hello;
    if ($foo == hello) {
        return 404;
    }
}
```

# Nginx 内置的全局变量

网上收集了一些 Ngixn 内置的全局变量，如下表所示：

| 变量名 | 含义 |
| --- | --- |
| $args | GET 请求中的参数 |
| $arg_PARAMETER | GET 请求中指定变量名参数的值，如 $arg_userName，获取到的是 GET 请求中 userName 参数的值 |
| $binary_remote_addr | 二进制码形式的客户端地址 |
| $body_bytes_sent | 传送页面的字节数 |
| $content_length | 请求头中的Content-length字段 |
| $content_typ | 请求头中的Content-Type字段 |
| $cookie_COOKIE | cookie  中指定的 COOKIE 的值 |
| $document_root | 当前请求在 root 指令中指定的值 |
| $uri | 请求中的当前 URI (不带请求参数，参数位于 $args)，不同于浏览器传递的 $request_uri 的值，它可以通过内部重定向，或者使用 index 指令进行修改，不包括协议和主机名 |
| $document_uri | 和 $url 相同 |
| $host | 请求中的主机头字段，如果请求中的主机头不可用或者空，则为处理请求的 server 名称 (处理请求的 server 的 server_name 指令的值)，值为小写，不包含端口 |
| $hostname | 机器名 |
| $http_HEADER | HTTP 请求头中的内容，HEADER 为 HTTP 请求中的内容转为小写，`-` 变为 `_` ( 破折号变为下划线 )，例如：$http_user_agent，$http_referer |
| $sent_http_HEADER | HTTP响应头中的内容，HEADER为HTTP响应中的内容转为小写，`-` 变为 `_` (破折号变为下划线)，例如： $sent_http_cache_control，$sent_http_content_type |
| $limit_rate | 连接速率 |
| $nginx_version | 当前运行的 Nginx 版本号 |
| $query_string | 与 $args 相同 |
| $remote_addr | 客户端的 IP 地址 |
| $remote_port | 客户端的端口 |
| $remote_user | 已经经过 Auth Basic Module 验证的用户名 |
| $request_filename | 当前连接请求的文件路径，由 roo t或 alias 指令与 URI 请求生成 |
| $request_body | 请求体 |
| $request_body_file | 客户端请求主体信息的临时文件名 |
| $request_completion | 如果请求成功，设为 "OK"，如果请求未完成或者不是一系列请求中最后一部分则设为空 |
| $request_method | GET 或 POST 请求方法 |
| $request_uri | 包含一些客户端请求参数的原始URI，它无法修改，请查看 $uri 更改或重写 URI |
| $scheme | 所用的协议，比如 http 或者是 https |
| $server_addr | 服务器地址，在完成一次系统调用后可以确定这个值，如果要绕开系统调用，则必须在 listen 中指定地址并且使用 bind 参数 |
| $server_name | 服务器名称 |
| $server_port | 请求到达服务器的端口号 |
| $server_protocol | 请求使用的协议，通常是HTTP/1.0或HTTP/1.1 |

参考资料：`http://www.nginx.cn/273.html`