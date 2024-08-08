---
date: 2024-08-08T16:29:18+08:00
tags:
  - docker
  - 容器
title: docker简介和安装
share: true
keywords: 
description: ""
lang: cn
cover:
  image: ""
author: 程元召
dir: posts
categories:
  - 云原生
---
## 概述
对于整个docker容器技术的体系来说，主要划分为三部分：镜像，容器和仓库
## 什么是镜像
在操作系统的体系，整个空间被横向切割为两大空间，内核空间和用户空间，在Linux系统启动的时候，内核启动后，会挂载一个root文件系统为用户空间提供支持，而镜像其实就是一个root文件系统，但是是一种特殊的root文件系统，提供包含了容器运行时所需要的程序，文件，资源，库，配置，还包含一些环境变量，用户等，镜像包含的全是静态的文件，在构建完成后，将不会改变。

如何形象的理解镜像？其实我们可以想象一下，镜像其实就是一个高层楼房，是有一层层叠加构成的。最初，什么也没有的时候就是空白镜像。后来我在空地上建了一层房子，并且分割好一个个房间，把东西分门别类的放进去，这就构成了我们的镜像，这里镜像可能是流行的发型版，如：红帽，乌班图等。后来，我发现我需要一个新的功能，但是一层已经建好，而且摆放好了，我只能再建一层，把我要的东西存储进去。所以整体看起来，镜像是分层的。这里就会存在一个问题，二层可能有三个住户，他们都想去一层修改东西，这就会导致大家读到的东西是不一致的，为了解决这个问题，一层就规定，我这一层是只能读的，不能写，你们要写，请复制一份到你们那里，自己修改。

![image](http://cdn.oyfacc.cn/docker/docker-image.jpg)

#### 镜像使用的技术
Union FS 联合文件系统是镜像技术使用的文件系统技术，为什么使用这个技术？镜像包含一个root文件系统，一般来说可能会比较大，所以使用分层存储的架构，ISO类型的镜像和这个是不一样的，这里的docker镜像其实是一个虚拟的概念，其最终的体现往往是一组文件系统的组合，而ISO一般是一个打包好的文件。
镜像build时一般是一层层生成，前一层是后一层的基础，本层build完就会固定，不会发生改变了，执行删除操作一般是标记为不可见，文件会一直存在，因此构建是尽量减少冗余文件的生成。

## 什么是容器
镜像和容器的关系，就像程序和进程的关系，容器就是运行时的镜像，容器可以被启动，停止，删除，暂停等，镜像一般只能被创建和删除。容器运行时会被添加一层可写层在最上边，用于容器运行时的数据增删改查，停止后会消失，不被保存，如何保存，会在后续的文章中说明。
&emsp;&emsp;容器运行的本质是进程,但和普通的进程是不一样的，他被添加了障眼法，运行在一个与世隔绝的空间内，也就是自己的名称空间内，就好像是一个独立的用户空间一样。空间内有自己的网络配置，用户，进程等。

![cont](http://cdn.oyfacc.cn/docker/continer.jpg)

## 什么是仓库
仓库，顾名思义就是存储镜像的地方，因为我们的镜像不是运行在一个主机上，在多个主机之间进行镜像的传输，就需要有一个存储和分发镜像的地方，就是仓库。一个仓库通常被永爱存储同一个镜像的不同版本，仓库被residtry统一管理。仓库名通常xx/xx:tag这种格式，用户名/服务名:tag,一般在私有仓库中往往是 地址or域名:端口/服务名:tag,这种格式，公有的docker registry是docker hub，私有的包含harbor或者nexus，或者官方的regisitry api。[Docker Hub 官方仓库地址](https://hub.docker.com/)

## docker结构分解

- [containerd](http://containerd.io/) is a container runtime which can manage a complete container lifecycle - from image transfer/storage to container execution, supervision and networking.
- container-shim handle headless containers, meaning once runc initializes the containers, it exits handing the containers over to the container-shim which acts as some middleman.
- [runc](http://runc.io/) is lightweight universal run time container, which abides by the OCI specification. runc is used by containerd for spawning and running containers according to OCI spec. It is also the repackaging of libcontainer.
- [grpc](http://www.grpc.io/) used for communication between containerd and docker-engine.
- [OCI](https://www.opencontainers.org/) maintains the OCI specification for runtime and images. The current docker versions support OCI image and runtime specs.

## 安装

生产环境中，我们经常需要安装指定版本的docker，稳定性和安全性方面都能得到保证。下面的教程是生产在用的一套教程，各位可以参考。

### 指定版本安装docker

```shell
NEW_USER=udocker
adduser $NEW_USER
echo "$NEW_USER ALL=(ALL) ALL" >> /etc/sudoers
export docker_version=18.06.3
yum install -y yum-utils device-mapper-persistent-data  lvm2 bash-completion
yum-config-manager --add-repo  http://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo
version=$(yum list docker-ce.x86_64 --showduplicates | sort -r|grep ${docker_version}|awk '{print $2}')
yum -y install --setopt=obsoletes=0 docker-ce-${version}
usermod -aG docker $NEW_USER
systemctl enable docker

```

### 锁定docker版本

```shell
yum install yum-plugin-versionlock
yum versionlock add docker-ce docker-ce-cli
yum versionlock list
yum versionlock delete <软件包名称>
yum versionlock clear
```

### 删除docker

```shell
yum remove docker \
              docker-client \
              docker-client-latest \
              docker-common \
              docker-latest \
              docker-latest-logrotate \
              docker-logrotate \
              docker-selinux \
              docker-engine-selinux \
              docker-engine \
              container*
```

##### 修改docker参数

```json
{
    "oom-score-adjust": -1000,
    "log-driver": "json-file",
    "log-opts": {
    "max-size": "100m",
    "max-file": "3"
    },
    "data-root": "/home/docker",
    "max-concurrent-downloads": 20,
    "max-concurrent-uploads": 10,
    "bip": "169.254.30.1/28", // 最好自定义一个完全不会冲突的网断
    "insecure-registries": [""], // 公司的镜像仓库
    "registry-mirrors": ["加速"],
    "storage-driver": "overlay2",
    "storage-opts": [
    "overlay2.override_kernel_check=true"
    ],
    "exec-opts": ["native.cgroupdriver=systemd"]
}

对于通过systemd来管理服务的系统(比如CentOS7.X、Ubuntu16.X), Docker有两处可以配置参数: 一个是docker.service服务配置文件,一个是Docker daemon配置文件daemon.json
```

### 修改服务单元参数

```shell
/usr/lib/systemd/system/docker.service
防止docker服务OOM： OOMScoreAdjust=-1000
开启iptables转发链：ExecStartPost=/usr/sbin/iptables -P FORWARD ACCEPT 
```

```
主机名配置
因为K8S的规定，主机名只支持包含 - 和 .(中横线和点)两种特殊符号，并且主机名不能出现重复
```

### 修改内核参数

```shell
net.bridge.bridge-nf-call-ip6tables=1
net.bridge.bridge-nf-call-iptables=1
net.ipv4.ip_forward=1
net.ipv4.conf.all.forwarding=1
net.ipv4.neigh.default.gc_thresh1=4096
net.ipv4.neigh.default.gc_thresh2=6144
net.ipv4.neigh.default.gc_thresh3=8192
net.ipv4.neigh.default.gc_interval=60
net.ipv4.neigh.default.gc_stale_time=120
# 参考 https://github.com/prometheus/node_exporter#disabled-by-default
kernel.perf_event_paranoid=-1
#sysctls for k8s node config
net.ipv4.tcp_slow_start_after_idle=0
net.core.rmem_max=16777216
fs.inotify.max_user_watches=524288
kernel.softlockup_all_cpu_backtrace=1
kernel.softlockup_panic=0
kernel.watchdog_thresh=30
fs.file-max=2097152
fs.inotify.max_user_instances=8192
fs.inotify.max_queued_events=16384
vm.max_map_count=262144
fs.may_detach_mounts=1
net.core.netdev_max_backlog=16384
net.ipv4.tcp_wmem=4096 12582912 16777216
net.core.wmem_max=16777216
net.core.somaxconn=32768
net.ipv4.ip_forward=1
net.ipv4.tcp_max_syn_backlog=8096
net.ipv4.tcp_rmem=4096 12582912 16777216
net.ipv6.conf.all.disable_ipv6=1
net.ipv6.conf.default.disable_ipv6=1
net.ipv6.conf.lo.disable_ipv6=1
kernel.yama.ptrace_scope=0
vm.swappiness=0
# 可以控制core文件的文件名中是否添加pid作为扩展。
kernel.core_uses_pid=1
# Do not accept source routing
net.ipv4.conf.default.accept_source_route=0
net.ipv4.conf.all.accept_source_route=0
# Promote secondary addresses when the primary address is removed
net.ipv4.conf.default.promote_secondaries=1
net.ipv4.conf.all.promote_secondaries=1
# Enable hard and soft link protection
fs.protected_hardlinks=1
fs.protected_symlinks=1
# 源路由验证
# see details in https://help.aliyun.com/knowledge_detail/39428.html
net.ipv4.conf.all.rp_filter=0
net.ipv4.conf.default.rp_filter=0
net.ipv4.conf.default.arp_announce = 2
net.ipv4.conf.lo.arp_announce=2
net.ipv4.conf.all.arp_announce=2
# see details in https://help.aliyun.com/knowledge_detail/41334.html
net.ipv4.tcp_max_tw_buckets=5000
net.ipv4.tcp_syncookies=1
net.ipv4.tcp_fin_timeout=30
net.ipv4.tcp_synack_retries=2
kernel.sysrq=1
```

