---
layout:     post
title:      Docker中搭建Jenkins
subtitle:
date:       2020-07-05
author:     parting_soul
header-img: img/持续集成01.jpg
catalog: true
tags:
    - Android
    - 持续集成
    - 笔记
---

### 一. docker 安装

升级内核

```shell
sudo yum update
```

安装需要的软件包

```shell
sudo yum install -y yum-utils device-mapper-persistent-data lvm2
```

设置yum源

```shell
sudo yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
```

安装docker

```shell
sudo yum install docker-ce
```

启动并加入开机启动

```shell
sudo systemctl start docker
sudo systemctl enable docker
```

验证安装是否成功

```shell
docker version
```

### 二. docker常用命令

#### 查看已经安装的镜像

```shell
docker images
```

#### 启动容器

```shell
docker start [容器名或者容器id]
```

#### 停止容器

```shell
docker stop [容器名或者容器id]
```

### 三. 安装Jenkins 

#### 3.1 下载最新版的docker镜像

```shell
docker pull jenkins/jenkins:lts;
```

#### 3.2 运行Jenkins容器

```shell
docker run -d --name jekins_server  -p 8080:8080 -p 50000:50000 -v /home/jenkins_home:/var/jenkins_home jenkins/jenkins
```

![image-20200629074242412](http://img.partingsoul.cn//image-20200629074242412.png)

参数说明：

- -d ： 后台运行
- --name : 为容器起一个名字
- -p ： 端口映射
- -v ：将宿主机的目录挂载到容器目录

#### 3.3 错误排查

查看容器是否运行成功过

```shell
docker ps
```

若发现容器未正常运行成功，可以查看容器的日志

```shell
docker logs [容器id]
```

![image-20200629074440429](http://img.partingsoul.cn//image-20200629074440429.png)

可以看到宿主机器被挂载到容器的目录没有访问权限，也就是/var/jenkins_home目录对应的真实目录/home/jenkins_home没有访问权限

![image-20200629075038537](http://img.partingsoul.cn//image-20200629075038537.png)

可以看到jenkins_home除了文件的所有者，其他组的用户没有写的权限

修改权限

```shell
chmod 777 [文件]
```

![image-20200629075339478](http://img.partingsoul.cn//image-20200629075339478.png)

由于之前启动过名字为jenkins_server这个容器了，虽然失败了，要再次启动，需要把之前的启动失败的容器移除掉

```shell
docker rm [容器id]
```

重新启动容器，通过ps命令查看，发现容器进程已经启动了

![image-20200629075703799](http://img.partingsoul.cn//image-20200629075703799.png)

#### 3.4 Jenkins 配置

服务启动了，可以配置Jenkins了

在浏览器输入Jenkins 服务的地址，我这边是 http://192.168.31.205:8080/

回车后，可以看到以下的界面

<img src="http://img.partingsoul.cn//image-20200629080246053.png" alt="image-20200629080246053" style="zoom:50%;" />

找到指定文件中Jenkins的初始密码

这边有两种方式可以查看到Jenkins的初始密码

- 在宿主机器挂载到容器的目录中找(我这边是/home/Jenkins_home目录)
- 进入容器内部，在上述图片中的文件路径下寻找

第一种方式不再赘述，这边讲一下第二种方式，进入容器内部

```shell
docker exec -it --user root [容器id] /bin/bash
```

![image-20200629095121266](http://img.partingsoul.cn//image-20200629095121266.png)

查看initialAdminPassword文件内容

```shell
cat initialAdminPassword 	
```

在输入管理员密码后，就可以进入Jenkins系统了

### 四. 系统配置

#### 4.1 安装JDK

首先需要下载JDK，然后需要将JDK复制到容器内部

```shell
docker cp [宿主文件路径] [容器名称]:[容器文件路径]
```

例如：

![image-20200629131856323](http://img.partingsoul.cn//image-20200629131856323.png)

配置JDK

![image-20200629140052028](http://img.partingsoul.cn//image-20200629140052028.png)

#### 4.2 安装Git

设置自动安装Git

![image-20200629140159958](http://img.partingsoul.cn//image-20200629140159958.png)

在管理Jenkins------系统配置中设置Git用户

![image-20200629152905301](http://img.partingsoul.cn//image-20200629152905301.png)

#### 4.3 安装Android  SDK

##### 1. 安装Android Tools

-  [下载Android Tools](https://dl.google.com/android/repository/sdk-tools-linux-3859397.zip?utm_source=androiddevtools&utm_medium=website)
- 将工具拷贝至Jenkins容器中

```shell
unzip sdk-tools-linux-3859397.zip 
docker cp tools/ jekins_server:/usr/local/Android/tools
```

##### 2. 下载指定版本的Android SDK

进入Android Tools的bin目录，使用sdkmanager查看可安装的列表

```shell
./sdkmanager --list
```

部分可安装列表截图

![image-20200629215350026](http://img.partingsoul.cn//image-20200629215350026.png)

下载指定版本的SDK

```shell
./sdkmanager "platforms;android-23" "platforms;android-24"  "platforms;android-25" "platforms;android-26" "platforms;android-28" "platforms;android-29"
```

安装platform-tools

```shell
./sdkmanager "platform-tools"
```

安装build-tools

```shell
./sdkmanager "build-tools;27.0.3" "build-tools;28.0.0"
```

若需要NDK，则安装NDK

```shell
./sdkmanager "ndk;21.3.6528147"
```

SDK授权(全部yes)

```shell
./sdkmanager --licenses
```

在Jenkins管理--系统配置中设置Android SDK和NDK的环境变量

![image-20200629222435267](http://img.partingsoul.cn//image-20200629222435267.png)

#### 4.4 安装Gradle

![image-20200629231427587](http://img.partingsoul.cn//image-20200629231427587.png)

到这里基本的配置就完成了

#### 4.5 构建项目

- 输入项目名称，选择自由风格的项目，点击OK

![image-20200629232145790](http://img.partingsoul.cn//image-20200629232145790.png)

- 添加项目仓库地址，配置好用户名密码

![image-20200629233807076](http://img.partingsoul.cn//image-20200629233807076.png)

- 选好gradle以及需要执行的task，这边是打包

![image-20200629233915901](http://img.partingsoul.cn//image-20200629233915901.png)

- 保存项目配置，点击Build Now开始构建项目

![image-20200629234018080](http://img.partingsoul.cn//image-20200629234018080.png)

- 查看构建日志

![image-20200629234048196](http://img.partingsoul.cn//image-20200629234048196.png)

选中进度条，进入任务构建界面

![](http://img.partingsoul.cn//image-20200703115218724.png)

点击控制台输出，即可查看构建日志

![image-20200703150950652](http://img.partingsoul.cn//image-20200703150950652.png)

