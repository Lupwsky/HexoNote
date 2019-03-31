---
title: SpringBoot-加载JAR包外面的页面和静态资源文件
date: 2019-03-10 22:25:33
categories: Spring
---

Spring Boot 将项目默认打包成 JAR 包后使用 java -jar xxx.jar 的方式就可以启动项目, 如果项目中有 HTML 页面以及静态资源文件, 也一并会打包到 JAR 包里面, 如果需要更改 HTML 页面和静态资源文件, 在更改后又需要将打包整个项目, 然后上传到服务器运行,  Spring Boot 支持将这些资源文件放到 jar 包外面并访问, 这样的又需要更改 HTML 页面和静态资源文件时只需要将更改后的资源文件上传到服务器, 然后将项目重启即可, 免去了整个项目打包和上传的过程

# 集成 Thymeleaf

为了后续方便测试, 在 Spring Boot 项目中集成 Thymeleaf, 添加如下依赖:

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-thymeleaf</artifactId>
</dependency>
```

<!-- more -->

接着在 application.properties 中添加如下 thymeleaf 配置:

```ini
spring.application.name=location
server.port=8880

### thymeleaf 相关配置 ###
spring.thymeleaf.prefix=classpath:/templates/
spring.thymeleaf.suffix=.html
spring.thymeleaf.mode=HTML
spring.thymeleaf.encoding=UTF-8
spring.thymeleaf.servlet.content-type=text/html

# 关闭缓存, 方便在本地调试, Build 一下即可即时刷新页面, 生产环境应改为 true
spring.thymeleaf.cache=false
```

在 resource 的 templates 目录下添加 hello.heml 和 error.html 页面, 里面的内容如下:

```html
<!DOCTYPE html>
<html xmlns="http://www.w3.org/1999/xhtml"
      xmlns:th="http://www.thymeleaf.org" lang="en">
<head>
    <meta charset="utf-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <meta name="renderer" content="webkit">
    <title>首页</title>
    <link rel="shortcut icon" th:href="@{/favicon.ico}"/>
</head>

<body style="overflow:hidden">
<div>
    正常访问页面~~, 嘿嘿
</div>
</body>
</html>
```

```html
<!DOCTYPE html>
<html xmlns="http://www.w3.org/1999/xhtml"
      xmlns:th="http://www.thymeleaf.org" lang="en">
<head>
    <meta charset="utf-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <meta name="renderer" content="webkit">
    <title>首页</title>
    <link rel="shortcut icon" th:href="@{/favicon.ico}"/>
</head>

<body style="overflow:hidden">
<div>
    您访问的页面不存在~~
</div>
</body>
</html>
```

接着定义 controller 类, 其中定义了一个访问错误页面时的跳转页面, 如下:

```java
@Controller
public class RouterController {

    @GetMapping(value = "/error")
    public String error() {
        return "/error";
    }

    @GetMapping(value = "/location/templates/test")
    public String templatesTest() {
        return "hello";
    }
}
```

启动项目, 输入 `localhost:8880/location/templates/test` 即可看到访问到的页面, 里面显示了 `正常访问页面~~, 嘿嘿` 内容, 接着在 resourdes 目录下的 static 放入一张 test.png, 接着访问 `localhost:8880/test.png` 就可以顺利的在浏览器看见图片

# 使用 JAR 包外部静态资源文件

Spring Boot 支持将这些资源文件放到 jar 包外面并访问, 先实现如何访问 static 目录下的静态文件, 在 application.properties 加入如下配置项, 配置多个可使用逗号分隔:

```ini
// (1) 当前 jar 包同一级的 static 目录可以这样配置 = file:./static
// (2) 使用 classpath 的其他目录的配置 = file:classpath:/other
// (3) 使用这个配置后就不能访问到 resource/static 下的静态文件了
spring.resources.static-locations=file:C:/Users/lpw/Desktop/static/,file:C:/Users/lpw/Desktop/static-bk/
```

将 test.png 文件放入, 并删除 resouorces/static 目录下的 test.png, 然后启动项目, 在浏览器输入 `localhost:8880/test.png` 就能访问到 test.png 了, 还可以设置能访问静态资源文件的 URL 匹配规则, 只有相匹配的才能访问到静态资源文件,  不匹配的则返回错误页面, 加入如下配置项, `这里不能配置多个, 配置多个无效`:

```ini
# /* 只匹配一级, /** 可匹配多级
spring.mvc.static-path-pattern=/images/**
```

上面实现的效果和使用如下代码配置是一样的效果的:

```java
@Configuration
public class MyWebAppConfigurer implements WebMvcConfigurer {

    @Override
    public void addResourceHandlers(ResourceHandlerRegistry registry) {
        registry.addResourceHandler("/images/**").addResourceLocations("file:C:/Users/lpw/Desktop/static/");
    }
}
```

`如果是前端页面想随时的修改静态文件, 个人建议是将这些静态文件放在文件服务器上, 页面中使用这些文件直接使用文件在文件服务器上的 URL 即可`

# 使用 JAR 包外部模板文件

## Thymeleaf 模板文件

将 Thymeleaf 模板文件配置到 jar 包的外面, 在网上没有找到配置的方法......, 似乎是不能配置的, 暂时不清楚, 在网上查找资料的时候, 找到一个解决使用 Thymeleaf 开发时跳转查看资源文件的问题方法, 可以用下面的写法解决,  :

```html
<script src="../static/js/html2canvas.min.js" th:src="@{/js/html2canvas.min.js}"></script>
<script src="../static/js/jquery-1.9.1.min.js" th:src="@{/js/jquery-1.9.1.min.js}"></script>
```

## Freemarker 模板文件

参考这篇文章里面的配置方法: [http://www.sandc.software/blog/how-to-load-freemarker-templates-from-external-file-system-folder-with-spring-boot/](http://www.sandc.software/blog/how-to-load-freemarker-templates-from-external-file-system-folder-with-spring-boot/)

```ini
# 热检测从文件系统加载的模板文件中的更改, 通常不需要设置, 因为默认的是 true
spring.freemarker.prefer-file-system-access=true
# 多个路径可以使用逗号分隔
spring.freemarker.template-loader-path=file:C:/Users/v_pwlu/Desktop/templates/
```