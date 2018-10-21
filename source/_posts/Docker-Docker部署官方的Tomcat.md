---
title: Docker-Docker部署官方的Tomcat
date: 2018-10-22 00:01:55
categories: Docker
---

# 在 Docker Hub 中搜索 Tomcat 镜像

```shell
docker search tomcat
```

![IMAGE](Docker-Docker部署官方的Tomcat/78FCDB13487FE3F52AB6951F4448660E.jpg)

<!-- more -->

# 开始安装

```shell
docker pull tomcat
```

![IMAGE](Docker-Docker部署官方的Tomcat/D2B05EA496B2238A0DD4504AF01BD623.jpg)

# 查看安装的镜像

```shell
docker images
```

![IMAGE](Docker-Docker部署官方的Tomcat/3872467C3BD2F1513F6F4F91E3E4944D.jpg)

# 运行 Tomcat

```shell
docker run -d -p 8081:8080 tomcat:latest (①：latest 是这个镜像的 tag 名称 ②：-d 参数可以其在后台运行)
```

![IMAGE](Docker-Docker部署官方的Tomcat/A8B3CF448D47328F58A8E7CCF22FE950.jpg)

其中 -p 8081:8080 表示指定端口号，并且将 8080 端口号映射到 8081 端口，启动成功后访问 `http://localhost:8081/` 就可看官方的测试页面，如下图：

![IMAGE](Docker-Docker部署官方的Tomcat/BA6605587F72A22E37E7F91E40911C67.jpg)