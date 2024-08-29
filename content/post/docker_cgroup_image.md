---
date: 2020-07-05T22:30:20+08:00
tags:
  - docker
  - 容器
title: Docker操作和资源隔离
share: true
keywords: 
description: ""
lang: cn
cover:
  image: ""
author: 程元召
dir: post
categories:
  - 云原生
  - 容器
draft: false
---
## 基本操作
容器启动之前，镜像必须先存在本地，如果本地不存在镜像，回去仓库获取镜像，并且pull的本地
### 获取镜像
获取镜像的命令是`docker pull`命令，命令格式如下：
```linux
docker pull [选项] [docker registry 地址/仓库名:标签]
```
具体的参数选项可以通过`docker pull --help`查看到。一般情况下获取镜像的命令是`docker pull 用户名/仓库名：tag`，如果没写用户名，则代表是官方的镜像库，如：`docker pull centos:7.4`,就是获取centos的7.4版本的镜像。上面的命令没有给出地址，所以默认从[docker hub](https://hub.docker.com/)获取。如果不写版本，则拉取标签为latest的镜像。
### 运行容器
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
### 列出镜像
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
### 删除镜像
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


## 资源隔离
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


Docker引爆了虚拟化领域对容器技术的关注，成为了现在最热门的技术之一。这边文章是对虚拟化技术的基础，名称空间进行总结。

#### 6大名称空间

在我的理解中，容器技术就像是一个正方体的盒子，有6个面组成，组成一个封闭的空间，这个6个面分别代表了内核系统调用的6个参数，也就是我们通常所说的6大名称空间。

- 主机名和域名  UTS                             CLONE_NEWUTS
- 信号量，共享内存，消息队列 IPC    CLONE_NEWIPC
- 进程号 PID                                          CLONE_NEWPID
- 网络 NET                                             CLONE_NEWNET
- 挂载点 MOUNT                                  CLONE_NEWNS
- 用户 USER                                           CLONE_NEWUSER

实际上上面的6大名称空间实在内核3.8版本以后才开始成熟的。所以最好升级内核版本，这样才稳定。

#### 内核系统调用

创建一个新的进程，大家都熟悉是使用系统调用`clone（）`，也是docker使用的方法。

他的基本用法如下：

```c
int clone(int (*child_func)(void *), void *child_stack, int flags, void *args)
```

flags就是控制使用多少功能，上面6中就包含在其中

child_func传入子进程的主函数

child_stack 传入子进程的栈

`sents（）`系统调用主要用于把进程加入一个已经存在的名称空间

```c
int sents(int fd, int nstype)
```

fd是要加入的名称空间描述符

nstype可以检查名称空间是否符合自己的要求，0代表不检查

`unshare()`系统调用是把原进程上进行隔离，不需要启动一个新的进程

`fork()`系统调用很多人比较熟悉，这里简单介绍下，fork本身去产生新进程会返回两次，一次是父进程，返回的是子进程的pid，子进程返回的是0，代表正确，负值代表错误。

### 示例

##### uts使用

```c
clone (child main, child_stack+STACK_SIZE, CLONE_NEWUTS| SIGCHID, NULL)
```

##### ipc使用

```c
clone (child main, child_stack+STACK_SIZE, CLONE_NEWIPC| SIGCHID, NULL)
```

##### pid使用

pid名称空间属于比较重要的隔离。内核首先创建了一个根名称空间，新的名称空间都是这个空间的子节点，构成一个树状结构，这样父节点可以看到子节点的进程，并且可以影响，反之不行。

![namespace](http://cdn.oyfacc.cn/namaspace.png)

在linux系统中，1号进程是上帝进程，一般负责管理和回收所有子进程，在名称空间中也一样，pid为1的进程对名称空间拥有特权，起特殊作用。kill命令无法影响父亲节点和兄弟节点。

通常情况下，容器内最好运行一个进程，但是实在有多个进程的需求，1号进程就很重要，必须有管理的功能，如bash。init进程有忽略信号的权利，就是不接受信号，除非编写了逻辑。但是父亲节点的信号kill和stop信号不可忽略，父亲节点有权停止子节点。

##### mount使用

历史上最早的名称空间，所以参数是CLONE_NEWNS，早起使用的是复制方法，就是把文件结构完整的复制一份给名称空间，但是存在一些问题：如无法自动挂载一些外部文件系统。

2006年引入的挂载传播解决了这个问题，挂载传播定义了对象之间的关系，从属关系和共享关系。

存在五种挂载状态

- 共享挂载，两个名称空间互相传播时间
- 从属挂载，父亲空间影响儿子空间，反之不行
- 共享/从属挂载，同时兼具两者的特征
- 私有挂载，互不影响
- 不可绑定挂载，互不影响

默认状态下，所有的挂载对象都是私有的。

其他名称空间基本使用基本类似。