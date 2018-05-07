---
title: SpringBoot-配置logback打印日志
date: 2018-04-08 14:30:26
categories:  SpringBoot
---

``` ini
# 配置日志
# 配置日志输出级别
logging.level.root=INFO
# 具体包名的的日志级别
logging.level.org.springframework.web=DEBUG
logging.level.org.hibernate=ERROR
# 默认情况下 Spring Boot 是不将日志输出到日志文件中，可以通过在 application.properites 文件中配置 logging.file 文件名称和 logging.path 文件路径，将日志输出到文件中
# 文件名不配置自动生成 spring.log 的文件，路径不存在会自动生成，前提是有配置日志的输出级别 logging.level.root，否则不会输出日志到文件中
# 可以只使用 logging.file，但是需要使用绝对路径
logging.file=log.log
logging.path=/Users/lupengwei/home/mw_wechat_mini/files
```

关于时区的问题： `https://segmentfault.com/q/1010000010791397`