---
date: 2024-08-08T18:29:18+08:00
tags: 
title: K8S基本概念
slug: 18:29
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
---
## node

`node`即节点，是物理服务器或者虚拟服务器，他们是组成kubernetes资源池的基础。节点分为两种色，`master`节点和`worker`节点。master节点负责资源调度，集群状态控制等，worker节点负责运行用户容器，承接负载。

##### 获取node信息

通过`kubectl`命令可以获取node信息

```shell
$ kubectl get nodes
NAME                     STATUS                     AGE
i-2ze0tfg75y5plzvnd29h   Ready,SchedulingDisabled   2d
i-2ze0woc5l1230xs5zxry   Ready                      2d
i-2ze14a3m7riw0l18oemg   Ready                      2d
i-2ze14a3m7riw0l18oemh   Ready                      2d
i-2ze1nwnt9tc3wg83rsru   Ready                      2d
```

获取更详细的信息可以使用如下命令

```shell
# kubectl get nodes -o wide
NAME                                 STATUS    ROLES     AGE       VERSION   INTERNAL-IP     EXTERNAL-IP   OS-IMAGE                KERNEL-VERSION              CONTAINER-RUNTIME
cn-shanghai.i-uf6143y5dc78k   Ready     master    1y        v1.11.5   172.18.88.44   <none>        CentOS Linux 7 (Core)   3.10.0-693.2.2.el7.x86_64   docker://17.6.2
cn-shanghai.i-uf61jtxho26   Ready     <none>    247d      v1.11.5   172.18.89.45   <none>        CentOS Linux 7 (Core)   3.10.0-693.2.2.el7.x86_64   docker://17.6.2

kubectl describe nodes  cn-shanghai.i-uf6143y5dc #描述node信息

#···信息太长这里就不展示了

```





# pod

`pod`是kubernetes非常重要的概念，他是运行应用的载体。有人说，pod是容器吗？等同吗？pod不是容器，pod更像是容器的封装体，如果把容器比喻为一个个的木箱子，那么pod就像是一个集装箱，一个集装箱可以装一个或多个木箱子，并且有挂钩这些方便吊装的装置。所以说pod是容器的封装，解决的是容器编排问题。

```shell
kubectl get pods #获取pod，不指定名称空间


kubectl get pods -n www #获取pod，指定名称空间，名称空间就是www

NAME                     READY   STATUS    RESTARTS   AGE
nginx-646b46d648-hbwg2   1/1     Running   0          101s

kubectl get pods -n www -o wide #获取pod运行于哪个节点
NAME                     READY   STATUS    RESTARTS   AGE     IP            NODE       NOMINATED NODE   READINESS GATES
nginx-646b46d648-hbwg2   1/1     Running   0          2m23s   172.17.0.11   minikube   <none>   

kubectl logs 646b46d648-hbwg2 #获取pod中的日志

kubectl exec -it nginx-646b46d648-hbwg2 /bin/bash #交互方式进入pod的内部

kubectl exec nginx-t213217u -- ls -l #对pod执行一个命令
# 这里的双横线（--）区分的是本地终端命令和容器中执行的命令，当中执行的命令只有一个单词时，可以忽略。

```


## Namespace

namespace， 名称空间，是一个虚拟的空间，主要是用来构建一个虚拟的资源成，将资源池划分成多个虚拟的区域，互不干扰。不通资源池的资源可以重名，但是名称空间本身不可以重名，相同的名称空间内的资源也不可以重名。

```shell
# kubectl get ns  #获取名称空间
NAME              STATUS   AGE
default           Active   27h
kube-node-lease   Active   27h
kube-public       Active   27h
kube-system       Active   27h
tutorial          Active   7s
```


## ReplicationController

一种k8s的资源类型，可以确保pod始终处于运行状态，如果减少则重新创建，增加则删除pod

RC主要有三个部分

label Selector： 标签选择器

Replica count：副本个数

pod template： pod 模板

这里需要注意的是标签选择器所选择的标签和pod模板内的标签必须是一致的，这样pod才会受RC控制。

## ReplicaSet

RC资源的进阶版，主要增强了RC的标签选择器功能，RC的标签选择器只支持等式选择，如：env=dev或者app=nginx。RS增强了这部分功能，支持env in dev or pro 这种写法，就是表达式写法，可以让我们有更灵活的使用。

## Deployment

一种资源类型，在生产环境中，通常使用更高级的资源来创建pod，不直接使用RC和RS，通常也不直接创建pod，而是使用副本控制器来创建pod，因为他能使pod维持在我们期望的一个状态，一旦有pod死掉，也会重新复制出来新的副本来维持我们所需要的状态。但是如果不使用副本控制器来创建，当前pod死掉，就真的死掉了，会直接影响业务。deployment是RS的高级封装，使用更友好。

```shell
[root@kong-ali-bj-001 ~]# kubectl run nginx --image=cargo.caicloud.io/caicloud/nginx:1.9.7 --replicas=1
kubectl run --generator=deployment/apps.v1 is DEPRECATED and will be removed in a future version. Use kubectl run --generator=run-pod/v1 or kubectl create instead.
deployment.apps/nginx created
上边提示的是建议使用，可以看到一个deployment被创建了
```

如何获取我们刚才创建的deployment

```linux
[root@kong-ali-bj-001 ~]# kubectl get deployment
NAME    READY   UP-TO-DATE   AVAILABLE   AGE
nginx   1/1     1            1           22s
可以获得刚才创建好的
```

## Label

标签在kubernetes是非常重要的概念，基本上各类控制器都要依赖标签来选择对应的资源，例如：deployment根据pod的标签来选择相对应的pod，污点也是根据标签来选择容忍的。

查看node的标签

```shell
[root@kong-ali-bj-001 ~]# kubectl get node --show-labels
NAME               STATUS   ROLES    AGE   VERSION   LABELS
kong-ali-bj-001    Ready    master   28d   v1.17.3   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/arch=amd64,kubernetes.io/hostname=kong-ali-bj-001,kubernetes.io/os=linux,node-role.kubernetes.io/master=
pgsql-ali-bj-001   Ready    <none>   28d   v1.17.3   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/arch=amd64,kubernetes.io/hostname=pgsql-ali-bj-001,kubernetes.io/os=linux
可以看到 beta.kubernetes.io/arch=amd64 这种类型的就是标签
```

查看pod的标签

 ```shell
[root@kong-ali-bj-001 ~]# kubectl get pod --show-labels
NAME                     READY   STATUS    RESTARTS   AGE     LABELS
myrepo-nwwn7             1/1     Running   0          28d     run=myrepo,test_label=pro
myrepo-qdhng             1/1     Running   0          28d     run=myrepo
nginx-6645f84fcd-q9294   1/1     Running   0          8m55s   pod-template-hash=6645f84fcd,run=nginx
 ```

对pod添加标签

```shell
[root@kong-ali-bj-001 ~]# kubectl label pod myrepo-nwwn7 tt=aa
pod/myrepo-nwwn7 labeled
label添加成功
 kubectl get pod --show-labels       
NAME                     READY   STATUS    RESTARTS   AGE   LABELS
myrepo-nwwn7             1/1     Running   0          28d   run=myrepo,test_label=pro,tt=aa
可以看到添加上去的lable
```

## configmap

k8s用来管理配置资源信息的一种资源类型。通过单独创建configmap，再将configmap挂载到pod内部，分离配置和应用。创建configmap可以从yaml文件和文件中创建，普通文件的格式类似如下：

```html
clor.good=true
map.ip=10.10.10.10
```

可以看到也是key-vlaue类型的

```shell
[root@kong-ali-bj-001 ~]# kubectl create configmap test --from-file=test.pro 
configmap/test created
从文件创建configmap
```

```shell
[root@kong-ali-bj-001 ~]# kubectl get configmap
NAME   DATA   AGE
test   1      40s
获取configmap
```

```shell
[root@kong-ali-bj-001 ~]# kubectl describe configmap test
Name:         test
Namespace:    default
Labels:       <none>
Annotations:  <none>

Data
====
test.pro:
----
color.good=true
allow=true
Events:  <none>
查看configmap的详细信息
```



 学习了一些基本的概念，我们开始一层层拆解K8S。学习任何一个主流的技术之前，一般都是从他的架构开始学习，这样可以比较全面的把握它的工作流程。

本片文章介绍如下内容：
1-什么是集群控制平面和节点
2-etcd的基础介绍
3-附件和插件机制

# 集群控制平面

集群控制平面，又叫做AKA master，也就是我们通常所说的master节点，通常是用于控制整个集群的运行状态。他可以单节点运行，也可以高可用运行，也可以托管于自身的集群上运行。支持REST API去CRUD集群资源。控制平面提供了集群唯一的入口，集群内的全部资源的操作，都要通过master节点来进行，提供了安全性的保证，可提供了便捷的操作。master节点主要包含如下的组件：

| 名称               | 主要功能                                                     |
| ------------------ | ------------------------------------------------------------ |
| API server         | k8s api服务，主要提供REST操作，验证资源和更新资源和etcd交互  |
| Contorller Manager | 其他集群功能的控制使用一个单独的进程来管理，就是控制管理器，控制某种资源的预期状态 |
| Scheculer          | 监控未调度的pod去绑定他们到主机，通过/binding的pod子api      |

### etcd

集群状态存储，api server所有的更新数据都要存储在etcd内部，同时etcd提供了watch的支持，就是当数据发生变化的时候回返回当前的数据，未发生变化时，阻塞watch的进程。

以上的组建主要是master节点所要依赖的组建，其中存储组建可以单独作为一个种类介绍

# 节点

这里的节点就是前文中的node，主要是k8s中的工作节点，主要负责运行pod。主要包含如下的组建：

| 名称               | 主要功能                                                     |
| ------------------ | ------------------------------------------------------------ |
| kubelet            | 主要负责运行调度到该节点上的pod，并且负责上报当前节点上的全部pod的运行状态 |
| containner runtime | 容器运行时，主要是把容器跑起来，拉取镜像。kubelet和容器运行时通过容器运行时接口来交互，接口目前支持：rkt，docker，cri-o，frakti |
| kube-proxy         | kubeproxy创建规则于iptables上，用于提供负载均衡和高可用解决方案，将流量转发到后端的pod上 |

# add-ons & plugins

附件，主要为集群提供扩展功能，目前主流的附件包含如下几个：

| 名称    | 作用                                                         |
| ------- | ------------------------------------------------------------ |
| DNS     | 提供解析功能和服务发现功能，用于集群内部的pod之间的通讯解析和查找 |
| network | 主要是集群之间的node进行通讯提供功能，主要是为每个节点分配一个网段的地址，每个pod调度后就可以获取到一个IP地址，通过ip地址进行通讯 |

# 总结

本片文章主要把一些基本的概念进行了一些简单的讲述，接下来的几篇文章会对几个流程进行讲解，主要包含如下几个：

1-master节点和node节点之间的通讯
2-调度器的工作机制
3-控制器的工作机制
4-kubelet运行机制
5-一个pod的完整运行流程总结