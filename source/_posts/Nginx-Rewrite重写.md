---
title: Nginx-Rewrite重写
date: 2018-05-04 00:17:29
categories: Nginx
---

Rewrite 的可以重新定向 URL 地址，通过在配置文件中重写 URL，可以让网站中的 URL 符合某种条件后跳转到指定的 URL，如重定向到 301，404 等页面，例如因为需求更改了项目的结构，用户之前收藏的链接再去访问就会出现错误，Rewrite 就可以用于解决这种问题。Rewrite 还可以完成防盗链的功能。源码的实现可以查看 ngx_http_rewrite_module 模块，官方地址 `https://nginx.org/en/docs/`。

# Rewrite 重写的语法

Rewrite 的重写规则既可以放在 service，也可以放在 location 里面。

## if
