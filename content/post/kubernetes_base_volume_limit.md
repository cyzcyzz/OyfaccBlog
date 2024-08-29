---
date: 2020-07-23T11:20:41+08:00
tags:
  - volume
title: K8S卷和资源限制
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
  - 容器
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

1. master节点和node节点之间的通讯
2. 调度器的工作机制
3. 控制器的工作机制
4. kubelet运行机制
5. 一个pod的完整运行流程总结


# 基本介绍

- 场景一：当我们用K8S创建了一个pod的时候，容器顶部附加了一个可写层用于程序运行时的数据读写，但在容器结束运行的时候，这个可写层也会随之消失，数据也会消失不见，数据无持久化。
- 场景二：每个pod都有自己的文件系统，当本次pod结束后，后创建的pod不能有效识别到前一个pod遗留的数据，数据不共享问题。

K8S通过定义存储卷来解决上述的问题。存储卷本身不是顶级资源，像pod， service那样，而是被定义为pod的一个组成部分，因为不是独立的K8S对象，不能独立的创建和删除，而且pod的所有容器均可共享使用这个卷。

下面一个简单的应用实例，第一个容器有htdocs目录和logs目录，存放html和日志文件，第二个容器运行了一个代理来创建html，第三个容器来收集日志

![volume-1](http://cdn.oyfacc.cn/k8s/vloume-1.png)

增加共享卷以后

![volume-2](http://cdn.oyfacc.cn/k8s/volume-2.png)

webserver写入日志，日志收集器来收集日志，两个容器内的文件系统的logs文件夹应该挂载同一个卷，htdocs和html文件夹，一个生成文件，一个读取文件，也应该使用同一个共享卷。

### 卷类型

K8S中有多种卷的类型，下面的列表可以参考：

| emptDir                                      | 存储临时数据的简单空目录               |
| -------------------------------------------- | -------------------------------------- |
| hostPath～                                   | 用于将目录从work                       |
| emptDir～                                    | 存储临时数据的简单空目录               |
| gitRepo                                      | 检出Git仓库内容来初始化                |
| nfs                                          | 挂载pod中的NFS共享卷                   |
| 谷歌高效存储卷 亚马逊弹性块存储 微软磁盘卷～ | 公有云存储                             |
| cinder cephfs glusterfs～                    | 其他网络类型的存储                     |
| configmap secret～                           | K8S部分资源和集群信息公开给pod的特殊卷 |
| PVC～                                        | 预置或者动态配置的持久存储类型         |

上面有波浪线的是使用类型比较多的卷，一个容器可以挂在多个不通类型的卷到不通的目录上。

### 使用示例

emptDir

```shell
apiVersion: v1
kind: Pod
metadata::
  name: fortune
spec:
  containers;
  - images; luksa/fortune
    name: html-gen
    volumeMounts:
    - name: html # 挂载的卷的名称
      mountPath: /var/htdocs # 这里指明容器内的挂在位置
      .......
   volumes:
   - name: html   # 这里定义了一个卷
     emptyDir: {}  
```

hostPath

这种类型属于比较常用的，我们生产线上都是挂载hostpath进去，写日志，ds filebeat去收集本机的日志到某一个日志机上

```shell
 volumeMounts:
 - mountPath: /home/logs # 挂载位置
   name: hostpath # 卷名称
      ....
      volumes:
        - name: hostpath #卷名称
          hostPath:
            path: /www-volume/logs # 主机路径
```

## PV&&PVC

之前的两种类型都需要pod的开发人员了解集群中的真实网络存储的基础设施，这对专业开发人员并不友好，理想的状况是，开发人员不需要知道这些底层设施，当我需要时，申请就可以了。 这就是pv和pvc的作用。

![pv](http://cdn.oyfacc.cn/2020-05-23-PVandPVC.png)

```shell
# 定义持久卷
apiVersion: v1
kind: persistentVolume
metadata:
  name: mongodb-pv
spec:
  capacity:
    storage: 1Gi  # 大小
  accessModes: # 可以被单个客户端挂载
  - ReadWriteOnce
  - ReadOnlyMany
  persistentVolumeReclimPolicy: Retain # 声明被释放后，pv将会保留
  gcePersistentDisk:
    pdName: mongodb
    fstype: ext4
```

持久卷不属于任何名称空间，是集群层面的资源，持久卷声明是属于某一个名称空间的资源，不能跨名称空间调用

```shell
# pvc
apiVersion: v1
kind: pVC #这里应该写全称
metadata: 
  name: monfodb-pvc
spec:
  resource:
    requests:
      storage: 1Gi
    accessModes:
    - ReadWriteOnce
    storageClassName: ""
    
    
...
volumes:
- name: mongodb-data
  persistentVolumeClaim:
    claimName: monfodb-pvc
```

删除持久卷：remian模式下需要手动释放

回收策略配置： recycle 和 delete将会自动释放持久卷

## storageclass

使用pv和pvc可以屏蔽底层的基础设施，但是仍但需要一个集群管理员来创建pv，为了解决这个问题，可以通过动态配置持久卷来自动自行次任务。

创建storageclass需要一个中间程序来连接我们的存储，无论是网络存储，云存储还是其他的存储。

```shell
apiVersions: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast
procisioner: kubenet. #这里是插件
parameters:
  type: pd
   # 传给插件的参数
  
  
 。。。
 pvc声明
 spec:
 storageClass:fast
...
```




k8s集群中，可以对两种资源进行限制，一是内存，二是cpu，使用Linux cGroups进行限制

### cpu限制

在Linux系统中，CPU的限制相比较内存的限制来说，更复杂一些。但是本身都可以通过cgroups来控制的，下面是一个K8S中的资源限制示例：

```yaml
resources:
  requests:
    memory: 50Mi
    cpu: 50m
  limits:
    memory: 100Mi
    cpu: 100m
```

单位后缀`m`表示千分之一核，也就是说，在K8S中，一个cpu核心被量化为1000m。上面的示例requests需要50m，也就是百分之五的cpu，要申请一个cpu的全部核心，可以使用1000m，或者1来表示。

这里调用docker来创建cgroups的时候，会有一些差异

Linux系统和docker都是按照1024的时间片值来计算的，这里和K8S有24的差值。我们称之为shares

shares用来设置CPU的相对值，如：a的shares值为1024，b的为512，则a的相对值为1024/1024+512 约为66%。shares有两个特点：

- a不忙，则b可以超过他的限制，a如果忙，则不能超过66的使用值
- 当有一个新的c 1024shares加入时，这个值重新计算。1024/1024+1024+512=40%，所以说这个值随着cgroups数量而改变的。

上面资源限制有两个字段，一个是requests，一个是limits。requests值，对应的就是我们上文介绍的shares值来计算的。limits又是怎么计算的呢？

其实他和requests使用的是不同的子系统来控制的。为什么会有单独的子系统呢。通过上文的介绍，你可能已经发现了，当一个进程没有设置shares的时候，他可以自由地使用cpu资源，而且shares子系统没法精准的控制使用的cpu，只能相对来控制。谷歌团队发现了这个问题，并且增加了一个子系统来控制：cpu带宽控制组。他定义了周期和配额两个属性。周期为1/10秒，100000微秒。配额为周期长度内可以使用的cpu时间数。两个配合起来可以精准的控制cpu的使用时长。周期值和配额均可以配置。

下面是几个例子：

```shell
# 1.限制只能使用1个CPU（每250ms能使用250ms的CPU时间）
$ echo 250000 > cpu.cfs_quota_us /* quota = 250ms */
$ echo 250000 > cpu.cfs_period_us /* period = 250ms */

# 2.限制使用2个CPU（内核）（每500ms能使用1000ms的CPU时间，即使用两个内核）
$ echo 1000000 > cpu.cfs_quota_us /* quota = 1000ms */
$ echo 500000 > cpu.cfs_period_us /* period = 500ms */

# 3.限制使用1个CPU的20%（每50ms能使用10ms的CPU时间，即使用一个CPU核心的20%）
$ echo 10000 > cpu.cfs_quota_us /* quota = 10ms */
$ echo 50000 > cpu.cfs_period_us /* period = 50ms */
```

上面例子中的100m,换算成这里就是10000/100000，默认的周期是100000，分别对应cfs_period_us=100000, cfs_quota_us=10000。

### 内存限制

内存限制相对于cpu就没有那么的复杂了。他直接把你设置的值映射到cgroups的内存控制子系统上。内存限制如上文的示例，requests代表你要申请的最少内存，也就是说，节点必须有这些内存可以申请，我可以超过这个内存，也可以少于这些内存，但节点一定要有。limits代表，我使用的内存上限，一旦超过，容器就要被oom了，所以建议两个设置合理的值。

### 默认限制

LimitRange 是用来设置 namespace 中 Pod 的默认的资源 request 和 limit 值，以及大小范围

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: mem-limit-range
  namespace: example
spec:
  limits:
  - default:  # default limit
      memory: 512Mi
      cpu: 2
    defaultRequest:  # default request
      memory: 256Mi
      cpu: 0.5
    max:  # max limit
      memory: 800Mi
      cpu: 3
    min:  # min request
      memory: 100Mi
      cpu: 0.3
    maxLimitRequestRatio:  # max value for limit / request
      memory: 2
      cpu: 2
    type: Container # limit type, support: Container / Pod / PersistentVolumeClaim
```

limitRange支持的参数如下：

- default 代表默认的limit
- defaultRequest 代表默认的request
- max 代表limit的最大值
- min 代表request的最小值
- maxLimitRequestRatio 代表 limit / request的最大值。由于节点是根据pod request 调度资源，可以做到节点超卖，maxLimitRequestRatio 代表pod最大超卖比例。