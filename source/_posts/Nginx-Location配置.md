---
title: Nginx-Location配置
date: 2018-05-03 22:14:22
categories: Nginx
---

location 是虚拟主机不可缺少的一个配置, 可以根据不同的 URI 定位到不同的处理方式上, 例如在碰到 .jsp、.html 为后缀的 URL 时做出相应的处理。

# Location 语法

常见的语法如下：
location [ = | ~ | ~* | ^~ | ! ] uri {
    ...
}

总共分成三个部分, location 关键字, 匹配修饰符和 uri, 其中匹配修饰符可以不写, 而 uri 可以为正则表达式, 但是原则上是能够尽量精确旧尽量精确表示, 匹配修饰符的含义见下表：

| 匹配修饰符 | 含义                                |
|-------|-----------------------------------|
| ~     | 区分大小写的正则匹配                        |
| ~!    | 不区分大小写的正则匹配                       |
| =     | 精准匹配                              |
| ^~    | 表示如果该符号后面的字符是最佳匹配, 采用该规则, 不再进行后续的查找 |
| @     | 内部跳转 |

<!-- more -->

# Location 匹配的类型

1. 精准匹配, 匹配修饰符为 = 号, 如 location = /get/user {}
2. 一般匹配, 不填写匹配修饰符, 如 location /get/user {}
3. 正则匹配, 填写其他非 = 号的修饰符, 如 location ~ /get/user {}

# Location 匹配的规则

1. 首先查看是否由精准匹配, 如果匹配成功, 则停止匹配过程, 匹配不成功就去匹配一般匹配;
2. 一般匹配可以理解为对字符串的正则匹配, 对多个一般匹配, 匹配的字符串越长就启用哪个匹配, 例如/foo, /get/user/foo/v1 时可以匹配上的, / 也可以匹配 /get/user/foo/v1, 但是 /foo 会发挥作用, 他匹配的字符串更长;
3. 一般匹配和正则匹配都生效了, 正则匹配发挥作用;
4. 多个正则匹配则第一个匹配的正则匹配发挥作用。

看一个例子, 在一个虚拟主机下面配置了两个 location, 如下, 当我们访问 `http://localhost/` 时, 无论我们怎么改 /user/lupw/html/ 目录下的 index.html, 看到的访问的内容总是 html 目录下的 index.html 问件里面的内容。为什么呢？是这样的, 访问 `http://localhost/` 时, 首先精准匹配到了第一个 location, 这个时候我们访问的 `http://localhost/` 时, 内部时转换成 `http://localhost/index.html` 后再进行访问, 因为第一个 location 的 index 设置是这样的, 且一个精准匹配无法访问一个目录,  / 只是一个目录, 最终会被解析去访问 index.html 文件, 会在内部会再次进行访问的 `http://localhost/index.html`, 会再进行匹配, 这个时候就会匹配到一般匹配了, 所以访问的是 html 目录下的 index.html 而不是  /user/lupw/html/ 目录下的 index.html。如果精准匹配匹配的是 location = /index.html, 访问的则是 /user/lupw/html/ 目录下的 index.html。

```sh
# 精准匹配
location =/ {
    root /user/lupw/html/;
    index index.html;
}

# 一般匹配, 这个例子也是一个通用匹配, 如果没有其它匹配,任何请求都会匹配到
location / {
    root html;
    index index.html;
}
```

# Location 内部跳转的例子

```sh
# 以 /img/ 开头的请求, 如果链接的状态为 404, 则会匹配到 @img_err 这条规则上。
 location /img/ {
    error_page 404 @img_err;
}

location @img_err {
    # 规则
}
```

# Location 的 URL尾部的 /

1. location 中的字符有没有 / 都没有影响, 也就是说 /user/ 和 /user 是一样的;
2. 如果 URL 结构是 `https://domain.com/` 的形式, 尾部有没有 / 都不会造成重定向;
3. 如果 URL 结构是 `https://domain.com/dir/`, 尾部如果缺少 / 将导致重定向, URL尾部的 / 表示目录, 没有 / 表示文件。所以访问 /dir/ 时, 服务器会自动去该目录下找对应的默认文件。如果访问 /dir 的话, 服务器会先去找 dir 文件, 找不到的话会将 dir 当成目录, 重定向到/dir/, 去该目录下找默认文件。

# Location 实际使用建议

```sh
# 直接匹配网站根, 通过域名访问网站首页比较频繁, 使用这个会加速处理, 官网如是说。
location = / {
    proxy_pass http://tomcat:8080/index
}

# 处理静态文件请求, 有两种配置模式, 目录匹配或后缀匹配, 任选其一或搭配使用
location ^~ /static/ {
    root /webroot/static/;
}
location ~* \.(gif|jpg|jpeg|png|css|js|ico)$ {
    root /webroot/res/;
}

# 转发动态请求到后端应用服务器
location / {
    proxy_pass http://tomcat:8080/
}
```