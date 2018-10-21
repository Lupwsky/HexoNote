---
title: Docker-在Mac上安装Docker
date: 2018-10-21 23:54:12
categories: Docker
---

# Mac 上使用 HomeBrew Cask 安装

安装指令：`brew cask install docker`

![IMAGE](Docker-在Mac上安装Docker/14F8C95A8433949C42265CBB127072FA.jpg)

<!-- more -->

下载太慢，我直接复制到迅雷直接下载下来了

![IMAGE](Docker-在Mac上安装Docker/E5A6ECBD3FEC3D2B604705297685C7C8.jpg)

# 检测 Docker 安装的版本

docker --version

# 更改镜像源

使用国内的网络拉取官方默认的镜像会很慢，可以配置国内的镜像源，这里配置网易的镜像地址：`http://hub-mirror.c.163.com` 
关于国内镜像源的选择参考：`https://ieevee.com/tech/2016/09/28/docker-mirror.html`

![IMAGE](Docker-在Mac上安装Docker/3252ADA783B1709F527B2D2122C95090.jpg)

配置好后重新启动 Docker 即可，使用 docker info 可以查看是否配置镜像源成功:

![IMAGE](Docker-在Mac上安装Docker/D2F5AA700F704C9590E21E7B61193612.jpg)

# 参考资料

* [http://www.runoob.com/docker/macos-docker-install.html](http://www.runoob.com/docker/macos-docker-install.html)
