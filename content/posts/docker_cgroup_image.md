---
date: 2024-08-08
tags: 
title: docker镜像和资源隔离
slug: 17:21
share: true
keywords:
  - 关键字1 - 关键字2
description: ""
lang: cn
cover:
  image: ""
author: 
dir: posts
categories:
  - 云原生
---
容器启动之前，镜像必须先存在本地，如果本地不存在镜像，回去仓库获取镜像，并且pull的本地
## 获取镜像
获取镜像的命令是`docker pull`命令，命令格式如下：
```linux
docker pull [选项] [docker registry 地址/仓库名:标签]
```
具体的参数选项可以通过`docker pull --help`查看到。一般情况下获取镜像的命令是`docker pull 用户名/仓库名：tag`，如果没写用户名，则代表是官方的镜像库，如：`docker pull centos:7.4`,就是获取centos的7.4版本的镜像。上面的命令没有给出地址，所以默认从[docker hub](https://hub.docker.com/)获取。如果不写版本，则拉取标签为latest的镜像。
## 运行容器
本地存在镜像以后，我们便可以尝试去运行镜像了，运行容器使用`docker run`命令，格式如下:
```linux
$ docker run -it --rm \
    ubuntu:18.04 \
    bash
    -it 以交互式方式运行
    --rm 退出停止删除
    bash 容器启动后运行的命令
```
可以看到一个和正常的系统一样的终端，就是我们运行起来的容器了。
## 列出镜像
一台安装有docker的机器上，使用`docker image ls`命令可查看全部已存在镜像，如下所示：
```linux
$ docker image ls
REPOSITORY           TAG                 IMAGE ID            CREATED             SIZE
redis                latest              5f515359c7f8        5 days ago          183 MB
...
ubuntu               latest              f753707788c5        4 weeks ago         127 MB
```
包含仓库名 标签 镜像id 创建时间 大小，请注意这里看到的镜像体积不一定是是实际的大小，因为镜像本身和分层构建的，可能这个镜像的前面几层适合别人复用的层。另外这里的镜像大小要比docker hub上大，这是因为docker hub上包含的镜像大小是压缩后的大小。
&emsp;&emsp;`docker image ls ubuntu`可以列出所有仓库名称为ubuntu的镜像，上述的镜像列表中可以看到一个《none》的镜像，这种镜像叫做虚悬镜像，产生这种镜像的原因一般是有新的镜像和这个镜像重复名称和tag导致原本的镜像名称和tag被占用，从而名称变成none，下面的命令可以专门的查看这类镜像`docker image ls -f dangling=true`
```linux
$ docker image ls -f dangling=true
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
<none>              <none>              00285df0df87        5 days ago          342 MB
```
一般来说这类镜像已经失去了价值，除了占空间已经无用，可以删除掉，使用`docker image prune`删除。
## 删除镜像
删除镜像使用`docker image rm`命令，一般删除的时候使用镜像ID来删除，docker image ls列出的IMAGE ID是这个镜像的短ID，`docker image ls --digests`列出的是镜像的长ID，删除镜像会产生两种类型的信息 untagged和deleted一种是标记取消，一种是镜像层删除，当镜像层不被其他的镜像标记或者依赖的时候，就会在执行删除操作的时候被删除，下面是一些演示：
```linux
$ docker image ls
REPOSITORY                  TAG                 IMAGE ID            CREATED             SIZE
centos                      latest              0584b3d2cf6d        3 weeks ago         196.5 MB
...
nginx                       latest              e43d811ce2f4        5 weeks ago         181.5 MB
镜像列表


$ docker image ls --digests
REPOSITORY                  TAG                 DIGEST                                                                    IMAGE ID            CREATED             SIZE
node                        slim                sha256:b4f0e0bdeb578043c1ea6862f0d40cc4afe32a4a582f3be235a3b164422be228   6e0c4c8e3913        3 weeks ago         214 MB

$ docker image rm node@sha256:b4f0e0bdeb578043c1ea6862f0d40cc4afe32a4a582f3be235a3b164422be228
Untagged: node@sha256:b4f0e0bdeb578043c1ea6862f0d40cc4afe32a4a582f3be235a3b164422be228

长格式删除
$ docker image rm 501ad78535f0
Untagged: redis:alpine
Untagged: redis@sha256:f1ed3708f538b537eb9c2a7dd50dc90a706f7debd7e1196c9264edeea521a86d
Deleted: sha256:501ad78535f015d88872e13fa87a828425117e3d28075d0c117932b05bf189b7
Deleted: sha256:96167737e29ca8e9d74982ef2a0dda76ed7b430da55e321c071f0dbff8c2899b
Deleted: sha256:32770d1dcf835f192cafd6b9263b7b597a1778a403a109e2cc2ee866f74adf23
Deleted: sha256:127227698ad74a5846ff5153475e03439d96d4b1c7f2a449c7a826ef74a2d2fa
Deleted: sha256:1333ecc582459bac54e1437335c0816bc17634e131ea0cc48daa27d32c75eab3
Deleted: sha256:4fc455b921edf9c4aea207c51ab39b10b06540c8b4825ba57b3feed1668fa7c7

短格式删除
$ docker image rm centos
Untagged: centos:latest
Untagged: centos@sha256:b2f9d1c0ff5f87a4743104d099a3d561002ac500db1b9bfa02a783a46e0d366c
Deleted: sha256:0584b3d2cf6d235ee310cf14b54667d889887b838d3f3d3033acd70fc3c48b8a
Deleted: sha256:97ca462ad9eeae25941546209454496e1d66749d53dfa2ee32bf1faabd239d38
仓库名和标签删除
```

镜像的基本操作就介绍到这里了，更详细的用法，请参考官方文档。


### 隔离简介
进程隔离后，本身会看不到其他的进程，此时会发生一个问题，独占资源，也就是说，我这个进程会把全部的资源全部跑满，其他的进程抢夺不到资源，这时候我们通常需要对隔离的进程进行一些资源的限制，这就是Linux 下的cgroups技术。

cgroups是control groups的缩写，是内核提供的可以限制和隔离进程组所使用的资源的机制。

基本概念介绍
- 任务，task，通常是指一个系统进程
- 控制组，是按照一个标准划分的一组进程，资源控制都是以控制组为单位的。一个进程可以加入到一个控制组，也可以从一个进程组迁移到另一个控制组。
- 层级，控制组可以组织成层级的形式，就像是一颗树。控制族群树上的子节点继承父节点的属性
- 子系统，一个子系统就是一个资源控制器，如cpu，mem等，代表着一种可以限制的资源。一个子系统附加到一个层级后，表示这个层级上的控制组全部受这个子系统控制。

### 原理

cgroups的本质是给系统进程上挂上钩子（hooks），当task运行的时候触发钩子上所附带的子系统进行检测，最终按照设置进行资源限制和优先级分配。

`/proc/cgroups`可以查看支持的子系统

cgroups提供了虚拟文件系统作为用户接口，要是用系统，必须先进行挂载，默认挂载`/sys/fs/cgroups`

### 规则

同一个层级结构可以挂载多个子系统 ，一个子系统只能附加到一个层级结构上

每次系统创建新层级时，该层级内的所有任务都是默认cgroups也就是root cgroups的初始成员，根层级时系统自动创建的

一个任务可以时多个控制组的成员，但是cgroups必须是不同的层级

父进程clone子进程时，子进程自动属于父进程所属的控制组，可以根据需要移出

### 命令

#### lssubsys

查看全部子系统

#### lscgroup

查看全部的控制组