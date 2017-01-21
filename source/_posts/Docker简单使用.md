
---
title: Docker简单使用
tag: 工具 docker
---

[史上最简单Android源码编译环境搭建方法](https://zhuanlan.zhihu.com/p/24633328)这篇分享介绍了借助Docker来编译Android源码，Docker可以直接把编译工具链和Ubuntu系统整体打包，保证了编译环境和官方的一致。
我用的macOS，之前尝试过编译AOSP，出现各种问题，采用虚拟机的方式也常常编译不过，看到可以用docker的新方式，打算尝试用一下。但是从我实际操作来看，docker在mac上是很慢的，并不比虚拟机快多少，在编译AOSP时也会造成卡死，因为在mac平台上docker是要运行在一个虚拟机上的。在尝试docker编译AOSP失败后，我最终还是用macOS编译了，最终成功烧录到nexus手机上。
虽然docker不适合我编译AOSP，但是作为一个操作系统级虚拟化实现方案，还是非常优秀滴。写一个编译器做成一个镜像，放在docker容器里运行还是绰绰有余的，比如《自制编译器》里的`cbc编译器`, 有人就做了一个镜像上传到[DockerHub](https://hub.docker.com/r/leungwensen/cbc-ubuntu-64bit/)上了，这样就避免了需要配置一堆环境依赖的麻烦了。
也算是对Docker研究了半天，简单记录下docker的用法吧。

---
## Docker简介

### 为什么说Docker比虚拟机快呢？
因为Docker容器需要的开销有限。和传统的虚拟化相比，容器运行不需要模拟层和管理层，而是使用操作系统的系统调用接口。这降低了运行单个容器所需的开销，也使得宿主机中可以运行更多容器。
但这个是对Host机为Linux而言的，macOS上运行docker容器本质上还是跑在linux虚拟机上的。

### 什么是docker
镜像是Docker生命周期中的构建或打包阶段，而容器则是启动或执行阶段。
简单来说，Docker就是：
* 一个镜像格式
* 一系列标准的操作
* 一个执行环境

镜像是基于联合文件系统的一种层式的结构，由一系列指令一步一步构建出来，例如：添加一个文件；执行一个名人；打开一个端口。
当从一个镜像启动容器时，Docker会在该镜像的最底层加载一个读写文件系统，我们想在Docker中运行的程序就是在这个读写层中执行的。
下图是Docker的文件系统层。
![Docker文件系统层](http://p1.bqimg.com/567571/d31fea71fb692f4d.png)

构建镜像最方便的做法是写一个Dockerfile文件，让docker自带的工具读它，然后生出一个镜像文件来。
Dockerfile使用基本的基于DSL语法的指令来构建一个Docker镜像，之后使用docker build命令基于该Dockerfile中的指令构建一个新的镜像。
每条指令都会创建一个新的镜像层并对镜像进行提交。Docker大体上按照如下的流程执行Dockerfile中的指令。
* Docker从基础镜像运行一个容器。
* 执行一条指令，对容器做出修改。
* 执行类似docker commit的操作，提交一个新的镜像层。
* Docker再基于刚提交的镜像运行一个新容器。
* 执行Dockerfile中的下一条指令，直到所有指令都执行完毕。

这里给两个简单的例子，可以自己研究下：
* [cbc-ubuntu-64bit](https://hub.docker.com/r/leungwensen/cbc-ubuntu-64bit/)
* [aosp](https://hub.docker.com/r/kylemanna/aosp/)

---
## 基本用法
去[Docker官网](https://www.docker.com/)下载对应操作系统的安装包后，就可以使用了。
打开 Kitematic, 可以在这里下载镜像，dockerhub的形式跟github很像，可以commit、pull等。
![](http://p1.bqimg.com/567571/9d49704a11c5ecad.jpg)

Docker是基于C/S架构的，它有一个docker程序，既能作为客户端，也可以作为服务端。作为客户端时，docker程序向Docker守护进程发送请求（如请求返回守护进程自身的信息），然后再对返回的请求结果进行处理。

### 通过docker info 可以得到的一些信息
```bash
$ docker info      
Containers: 10
 Running: 1
 Paused: 0
 Stopped: 9
Images: 7
Server Version: 1.12.6
//...
Registry: https://index.docker.io/v1/
WARNING: No kernel memory limit support
Insecure Registries:
 127.0.0.0/8
```
### 创建交互式容器
我们告诉Docker执行`docker run`命令，我们告诉Docker基于`ubuntu`镜像来创建容器，如果本地没有该镜像的话，Docker会连接官方维护的Docker Hub Registry查找该镜像，下载并保存到本地宿主机中。 `-i`保证容器中STDIN是开启的，`-t`告诉Docker为要创建的容器分配一个伪tty终端。这样，新创建的容器才能提供一个交互式shell。最后的`/bin/bash`告诉Docker在新容器中要运行什么命令。其中`--name`参数告诉Docker创建一个名为`test_container`的容器。
```bash
$ sudo docker --name test_container run -i -t ubuntu /bin/bash
//...
root@12345:/#
```
这样，我们就能看到容器内的shell了。容器的id是12345。容器的主机名就是该容器的ID。具体可以通过`cat /etc/hosts`查看。
输入`exit`,就可以返回宿主机的命令行了。一旦退出容器，`/bin/bash`命令也就结束了，容器也随之停止运行。但容器是仍然存在的。
```bash
root@12345:/# exit
```
用`docker ps -a`命令查看当前系统中容器的列表

Docker容器重新启动的时候，会沿用docker run命令时指定的参数来运行。
```bash
$ sudo docker restart (container name or id)
```
---
### 创建守护式容器
`-d`参数告诉Docker把容器放到后台运行。
```bash
$ sudo docker run --name daemon_dave -d ubuntu /bin/sh -c "while true; do echo hello world; sleep 1; done"
```

### 容器内部都在干些什么
用`docker logs`命令来获取容器的日志。
```bash
$ sudo docker logs (container name or id)
```

### 查看容器内的进程
```bash
$ sudo docker top (container name or id)
```

### 在容器内部运行进程

通过`docker exec`命令在容器内部额外启动新进程，
如下启动了新的后台任务和交互式任务。
```bash
$ sudo docker exec -d (container name or id) touch /etc/new_config_file
```
```bash
$ sudo docker exec -t -i (container name or id) /bin/bash
```

### 停止守护式容器
```bash
$ sudo docker stop (container name or id)
```

### 自动重启容器
`--restart`标志会检查容器的退出代码，并据此来决定是否重启容器。
比如`--restart=onfailure:5`表示Docker会尝试自动重启该容器，最多重启5次。
```bash
$ sudo docker run --restart=always --name daemon_dave -d ubuntu /bin/sh -c "while true; do echo hello world; sleep 1; done"
```

### 深入容器
`docker inspect`命令会对容器进行详细的检查，然后返回其配置信息，包括名称、命令、网络配置等。这是一大串json数据，可以用`--format`标志来选定查看结果。
```bash
$ sudo docker inspect --format='{{.Args}}' (container name or id)
[run.sh docker]
```

### 删除容器
```bash
$ sudo docker rm (container name or id)
```
一次删除所有容器
```bash
$ sudo docker rm `docker ps -a -q`
```

### 列出镜像
用`docker images`得到本地的镜像列表。
```bash
$ sudo docker images
REPOSITORY  TAG     IMAGE ID       CREATED    	SIZE
ubuntu      latest  104bec311bcd   4 weeks ago  129 MB
ubuntu      14.04   xxx   		   x weeks ago  xxx MB
```
镜像保存在仓库中，而仓库在于Registry中，默认的Registry是Docker Hub。
每个镜像仓库都可以存放很多镜像（比如ubuntu仓库包含了ubuntu各个版本的镜像）。
执行`sudo docker pull ubuntu`命令来拉取ubuntu仓库中所有内容。
每个镜像在列出来时都带有一个标签，用于对组成特定镜像的一些镜像层镜像标记。
用如下的方式来指定该仓库的某一镜像
```bash
$ sudo docker run -ti ubuntu:12.04 /bin/bash
```
Docker Hub中有两种类型的仓库：用户仓库和顶层仓库。
`用户名/仓库名`这种形式表示用户仓库，是由Docker用户创建的；
顶层仓库只包含仓库名部分，由Docker内部人来管理的。

---
##  后记
因为我只是想用docker来配一个编译aosp的环境，对于Docker的很多高级功能还没有接触，比如利用连接和卷之类的Docker特性来组合并管理运行与Docker中的应用、创建多容器的应用栈等。

 ---
 相关链接
[史上最简单Android源码编译环境搭建方法](https://zhuanlan.zhihu.com/p/24633328)
[cbc-ubuntu-64bit](https://hub.docker.com/r/leungwensen/cbc-ubuntu-64bit/)
[用 Docker 快速配置前端开发环境](http://numbbbbb.com/2016/09/26/20160926_%E7%94%A8%20Docker%20%E5%BF%AB%E9%80%9F%E9%85%8D%E7%BD%AE%E5%89%8D%E7%AB%AF%E5%BC%80%E5%8F%91%E7%8E%AF%E5%A2%83/)
[《第一本Docker书》](https://book.douban.com/subject/26285268/)
