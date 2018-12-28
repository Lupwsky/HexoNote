---
title: Docker-使用Docker部署SpringBoot项目
date: 2018-12-28 23:08:05
categories: Docker
---

前两天使用 docker-maven-plugin 插件制作了一个 SpringBoot 项目的 docker 镜像文件, 并进行了部署, 做一下笔记, 我的环境是在 windows 系统上使用虚拟机创建了 centerOS 系统, 然后在  centerOS 上安装 Docker, 设置并开放了 Docker 远程 API 调用的 IP 和端口号, 在 windows 上使用 docker-maven-plugin 插件制作镜像并远程部署到 Docker 上, maven docker 的插件有多种, 在此是用的是  docker-maven-plugin, 一个开源的插件, 代码和使用说明都托管在 github 上, [https://github.com/spotify/docker-maven-plugin](https://github.com/spotify/docker-maven-plugin)

# Docker 远程连接设置

```shell
# 找到 docker.service 文件进行编辑
sudo vim /usr/lib/systemd/system/docker.service

# 找到如下内容并更改如下
# ExecStart=/usr/bin/dockerd -H unix://
ExecStart=/usr/bin/dockerd -H tcp://0.0.0.0:2375 -H unix://var/run/docker.sock  

# 重新加载配置文件并重启 Docker
sudo systemctl daemon-reload
sudo systemctl restart docker

# 记得防火墙开放 2375 端口
sudo firewall-cmd --zone=public --add-port=2375/tcp --permanent
sudo firewall-cmd --reload
```

如果是 windows, 如下步骤进行端口暴露 `Setting -> General -> 勾选 Expose daemon on tcp://localhost:2375 without TLS`

<!-- more -->

# SpringBoot 项目中的插件的配置

```xml
<plugin>
    <groupId>com.spotify</groupId>
    <artifactId>docker-maven-plugin</artifactId>
    <version>1.2.0</version>
    <configuration>
        <imageName>springboot/${project.artifactId}</imageName>

        <!-- Dockerfile 文件的配置 -->
        <dockerDirectory>src/main/docker</dockerDirectory>
        <forceTags>true</forceTags>
        <imageTags>
            <imageTag>latest</imageTag>
        </imageTags>
        <resources>
            <resource>
                <targetPath>/</targetPath>
                <directory>${project.build.directory}</directory>
                <include>${project.build.finalName}.jar</include>
            </resource>
        </resources>

        <!-- docker 远程 API 地址-->
        <dockerHost>http://192.168.80.131:2375</dockerHost>
    </configuration>
</plugin>
```

# 创建 Dockerfile

```dockerfile
FROM openjdk:8-jdk-alpine
VOLUME /tmp

# 设置工作目录后, 日志文件才能顺利的保存在 /usr/local/app/logs 目录下
WORKDIR /usr/local/app

# 时区设置, 使镜像的时区和宿主机的时区一样, 如果不设置时区, 容器和宿主机会有 8 个小时的时间差
ENV TZ=Asia/Shanghai
RUN ln -snf /usr/share/zoneinfo/$TZ /etc/localtime && echo $TZ > /etc/timezone

COPY demo-1.0.jar app.jar
ENTRYPOINT ["java", "-Djava.security.egd=file:/dev/./urandom", "-jar", "app.jar"]
```

# 开始创建镜像

```shell
mvn clean package -DskipTests docker:build
```

# 使用镜像创建容器并启动容器

查询镜像, 验证镜像是否制作成功:

```shell
sudo docker images
```

可以看到已经成功的制作镜像, 输出结果如下:

```text
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
springboot/demo     latest              2615a7504ff3        29 hours ago        121MB
openjdk             8-jdk-alpine        04060a9dfc39        7 days ago          103MB
```

接着使用制作的镜像启动一个容器, 启动的指令如下:

```shell
sudo docker run --name web -d -p 8080:8080 -v /home/lupw/Web/logs:/usr/local/app/logs springboot/demo
```

使用 `sudo docker ps` 就可以看见容器被创建并已经顺利的启动了

