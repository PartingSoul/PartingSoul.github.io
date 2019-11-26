---
layout:     post
title:      容器
subtitle:
date:       2019-11-25
author:     parting_soul
header-img: img/post_bg.jpg
catalog: true
tags:
    - Java
---


[TOC]

#### 1. 容器

产生背景： 操作系统的不同的进程会在资源紧缺的情况下抢占资源，进程之前可能会有影响

目的： 一个独立的用户空间，为进程运行提供环境以及隔离其他进程，并且可以通过容器对容器所需的资源进行限制。

#### 2. Docker

docker主机(Host)： 安装了Docker程序的机器(Docker 直接安装在操作系统之上 )

docker客户端(Client)： 连接Docker主机进行操作

docker仓库：用于存放各种打包好的docker镜像

docker镜像： 软件打包好的镜像文件

docker容器：镜像启动后的实例称为一个容器

##### 使用步骤

- 安装Docker
- 去Docker仓库寻找软件的镜像
- 使用Docker运行这个镜像，这个运行的镜像称为容器
- 停止容器的运行就是停止软件的运行

```shell 
yum install docker //安装docker
systemctl start docker // 启动docker
systemctl enable docker //将docker服务设置为开机启动
systemctl stop docker //停止docker
docker search mysql //搜索mysql镜像
docker pull mysql:[tag] //没有标签默认下载最新的镜像
docker rmi [镜像id] //删除镜像
docker images //查看镜像
```

##### 容器操作

软件镜像---> 安装镜像 -> 产生一个容器

```shell
docker run --name [自定义的容器名字] -d [镜像名:[tag]] //运行指定的镜像 -d表示后台运行
docker ps //查看运行的容器
docker rm [容器id] //删除容器
docker run -d -p 8888:8080 tomcat //运行时进行端口映射，将虚拟机的8080端口映射到主机的8888
docker logs [containerName]/[containerTag] //查看容器的日志
```

