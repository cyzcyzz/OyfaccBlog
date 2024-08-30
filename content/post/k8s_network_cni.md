---
title: K8S网络和CNI
date: 2020-11-08T18:32:39+08:00
description: ""
tags: 
categories:
  - 云原生
dir: post
share: true
---

Kubernetets社区对网络实现方案开放了CNI标准，在我们部署K8S的时候，其中有一个步骤是安装CNI包，这里就是会把CNI插件所需要的基础组建的二进制文件给安装到`/opt/cni/bin`，包含如下组建：

```shell
# pwd
/opt/cni/bin
# ll -h
-rwxr-xr-x 1 root root 2.9M Mar 26  2019 bridge
-rwxr-xr-x 1 root root 7.3M Mar 26  2019 dhcp
-rwxr-xr-x 1 root root 2.1M Mar 26  2019 flannel
-rwxr-xr-x 1 root root 2.2M Mar 26  2019 host-device
-rwxr-xr-x 1 root root 2.2M Mar 26  2019 host-local
-rwxr-xr-x 1 root root 2.6M Mar 26  2019 ipvlan
-rwxr-xr-x 1 root root 2.2M Mar 26  2019 loopback
-rwxr-xr-x 1 root root 2.6M Mar 26  2019 macvlan
-rwxr-xr-x 1 root root 2.5M Mar 26  2019 portmap
-rwxr-xr-x 1 root root 2.9M Mar 26  2019 ptp
-rwxr-xr-x 1 root root 1.9M Mar 26  2019 sample
-rwxr-xr-x 1 root root 2.1M Mar 26  2019 tuning
-rwxr-xr-x 1 root root 2.5M Mar 26  2019 vlan
```



上边插件类型基本会分为三种类型

##### 第一类，main插件，用来创建具体的网络设备的二进制文件

如： bridge（网桥设备），ipvlan，loopback（回环设备），ptp（veth pair设备），vlan

##### 第二类，IPAM设备，地址管理设备，负责分配IP地址的二进制文件

如：dhcp，动态分配IP，host-local预先分配的ip

##### 第三类，CNI社区维护的内置CNI插件

如：flannel，专门为flannel项目提供的CNI插件；tuning，通过sysctl调整网络参数的设备；portmap，通过iptables配置端口设备的文件

上边列出了安装默认带的基本插件，像calico这种，需要先下载后，放到这个目录就可以。要实现网络解决方案，除了上边的CNI网络插件，还要有方案本身的实现。

##### 网络解决方案

这一部分主要是实现配置隧道设备，配置宿主机路由，配置宿主机ARP和FDB表信息等工作。如flanneld进程

接下来就是安装flanneld本身的安装，可以使用Damonset安装，也可以使用二进制文件直接安装，安装完成后并且启动后，会在每台宿主机上生成一个配置文件，如下面所示：

```shell
# cat /etc/cni/net.d/10-flannel.conflist
{
  "name": "cbr0",
  "plugins": [
    {
      "type": "flannel",
      "delegate": {
        "hairpinMode": true,
        "isDefaultGateway": true
      }
    },
    {
      "type": "portmap",
      "capabilities": {
        "portMappings": true
      }
    }
  ]
}
```

**这个文件是给容器运行时读取的**，可以看到插件里面有两个，容器运行时会按照顺序加载。理论上这个目录只存在一个配置文件，如果存在多个，会按照配置文件的首字母首先加载第一个配置文件。

#### CNI插件工作原理

当kubelet watch到pod被调度到本机后，就会开始进行创建pod的操作，他第一个创建的容器肯定是infra容器，这里创建容器是容器运行时，也就是dockershim组建，接着kubelet会执行setuppod方法，这个方法就是准备CNI参数，调用CNI插件准备网络。

参数分为两部分

- **dockershim的CNI环境变量**

- **dockershim从CNI配置文件加载到的配置信息**

变量部分最重要的CNI_COMMAND,有add方法和del分别对应添加和删除

配置信息是dockershim需要传入CNI的部分，以json格式传输，这部分被称之为网络配置信息，传给CNI插件后，插件会修改信息，补充信息，然后调用bridge，会传入上边的两部分信息，接下来briage插件会真正的开始创建虚拟设备，分别插入网桥和容器的一端

接下来bridge插件调用ipam插件，为设备分配ip地址，同时设置默认路由。

接下来为cni0网桥设置ip地址，操作完成后，cni插件会把容器地址返回给dockershim，然后被kubelet添加到pod status，这些流程基本就是二层叠加类型的创建过程了。

![cni](http://cdn.oyfacc.cn/cni%E8%B0%83%E7%94%A8.png)


### network namespace

Linux network namespace 是虚拟化网络的基石，所以我们首先介绍他。于Linux内核2.6版本引入，主要是隔离linux系统的设备，以及ip地址，端口，路由表，防火墙规则等网络资源。

通过命令我们可以设置网卡到新创建的名称空间中，然后可以互相ping通，但是路由表和防火墙规则还是隔离的

注意，创建的虚拟设备最多只能存在一个名称空间中

namespace存在的奥秘/proc/PID/ns

每个进程都有属于自己的/proc/pid/ns

进来看下，其实可以看到很多符号连接，其中一个作用就是某两个进程是否属于一个namespace

还有一个作用就是 文件描述符处于open状态，namespace就一直存在，也就是说不需要有进程也会存在

```
touch /tmp/net
mount --bind /proc/$$/ns/net /tmp/net
```

如上所示，只要挂载起来就能保持一致存在，知道net被卸载

3. namespace添加进程 setns 系统调用
4. 脱离namespace unshare系统调用

### veth pair

虚拟以太网卡，设备总是以成对方式出现，类似管道。

容器和虚拟以太网卡的关系，容器内的eth0和宿主机上一个veth是成对的关系

```
cat /sys/class/net/eth0/iflink

# 查找宿主机网卡
#!/bin/bash
echo -e "----statrt-------"
cd /sys/class/net/
for i in `ls |grep veth`;do
  index=$(cat $i/ifindex)
  if [[ ${index} == ${1} ]];then
       echo -e "${i}\n"
       echo -e "${index}"
  fi
done
echo -e "----------end--------"
```

### bridge

用于连接网卡设备，网桥是一个虚拟的二层设备，根据MAC地址转发到对应的端口。

```
ip link add name br0 type bridge
ip link set br0 up
# brctl  创建
brctl addbr br0
```

假设物理网卡是1.2.3.4， 网关是1.2.3.1

```
# 创建一对设备
ip link add veth0 type veth peer name veth1
ip addr add 1.2.3.101/24 dev veth0
ip addr add 1.2.3.102/24 dev veth1
ip link set veth0 up
ip link set veth1 up
# 设置到br0上
ip link set dev veth0 master br0
# 查看网桥有什么设备
bridge link
```

如上操作后，veth0和协议栈的连接就变成了单向通道，此时协议栈可以发送数据给veth0，但是veth0不能发送到协议栈，这就相当于，br0做了一层拦截，所以我们把ip应该配置给br0

```
ip addr del 1.2.3.101/24 dev veth0
ip addr add 1.2.3.101/24 dev br0
## 物理网卡连接到br0
ip link set dev eth0 master br0
## 此时eth0间、相当于一条网线了，所以去除ip
ip addr del 192.168.  dev eth0
## 设置默认路由
ip route add default via 192.168.3.1
```

### vxlan

隧道网络代表是vxlan，一种overlay技术，是基于三层网络构建二层网络。不同于一对一的隧道协议，vxlan是一个一对多的网络，一个vxlan设备可以像网桥一样自动学习其他对端ip地址，也支持静态转发表。

vxlan重要的概念：

- VTEH设备，网络边缘设备，用来进行vxlan的封包和解包工作
- VNI vxlan网络标识
- tunnel 隧道一个逻辑概念，一个虚拟隧道

### kubernetes网络

在kubernetes网络模型中，每台node节点有自己独立的网段呢，容器直接按照目标容器ip地址进行访问。所以重点解决如下问题：

- 各宿主机容器ip段不能重复，需要ip段分配机制
- pod发出的流量到达所在服务器时，服务器网络层应当根据目标ip转发到目标服务器的能力

所以重点就在，ip地址分配和路由

### CNI

是kubernetes与低层网络的一个抽象层，为k8s屏蔽了底层网络实现的复杂度，通识届欧了k8s具体网络插件的实现。

CNI主要有两类接口

```
Addnetwork(net *networkconfig, rt *runtimeconf) (result, error)
DelNetwork(net *networkconfig, rt *runtimeconf)
```

kubelet和cni给出了两个默认的文件系统路径 

- /etc/cni/net.d/ 配置文件
- /opt/cni/bin/

#### 主机内组网模型

veth pair + bridge将容器与主机的网络协议栈连接起来，从而使数据包可以进出pod。容器放在主机根network namespace 中连接到网桥，可以使用pod内之间互相通讯。

#### 跨节点组网模型



