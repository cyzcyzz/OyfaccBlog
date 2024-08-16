---
date: 2024-08-12T17:44:36+08:00
tags:
  - etcd
title: 安装ETCD高可用集群
slug: 17:44
share: true
keywords: 
cover:
  image: 
author: 程元召
dir: post
categories:
  - 云原生
---
## 环境规划



| hostname          | 名称     | IP地址           |
| ----------------- | ------ | -------------- |
| etcd01.oyfacc.com | etcd01 | 192.168.158.13 |
| etcd02.oyfacc.com | etcd02 |                |
| etcd03.oyfacc.com | etcd03 |                |
## 基础环境准备

```shell
# 免密钥认证
ssh-keygen
ssh-copy-id root@xxx

# 所有机器创建文件夹
mkdir /etc/etcd 证书存放目录
mkdir /var/lib/etcd 数据和wal存放目录
      /usr/bin/ 二进制存放
```



## 证书

根据认证对象可以将证书分成三类：服务器证书`server cert`，客户端证书`client cert`，对等证书`peer cert`(既是`server cert`又是`client cert`)：

`etcd` 节点需要标识自己服务的`server cert`，也需要`client cert`与`etcd`集群其他节点交互，当然可以分别指定2个证书，为方便这里使用一个对等证书

### 下载cfssl组件
```shell
]# wget https://github.com/cloudflare/cfssl/releases/download/v1.6.4/cfssl_1.6.4_linux_amd64

]# wget https://github.com/cloudflare/cfssl/releases/download/v1.6.4/cfssljson_1.6.4_linux_amd64

]# wget https://github.com/cloudflare/cfssl/releases/download/v1.6.4/cfssl-certinfo_1.6.4_linux_amd64

]# mv cfssl_1.6.4_linux_amd64 /usr/local/bin/cfssl

]# mv cfssljson_1.6.4_linux_amd64 /usr/local/bin/cfssljson

]# mv cfssl-certinfo_1.6.4_linux_amd64 /usr/local/bin/cfssl-certinfo
```

### CA配置
```json
# ca-config.json

{
  "signing": {
    "default": {
      "expiry": "876000h"
    },
    "profiles": {
      "kubernetes": {
        "usages": [
            "signing",
            "key encipherment",
            "server auth",
            "client auth"
        ],
        "expiry": "876000h"
      },
      "kcfg": {
        "usages": [
            "signing",
            "key encipherment",
            "client auth"
        ],
        "expiry": "876000h"
      }
    }
  }
}
```
### CA请求
```json
ca-csr.json


{
  "CN": "kubernetes-ca",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "HangZhou",
      "L": "XS",
      "O": "k8s",
      "OU": "System"
    }
  ],
  "ca": {
    "expiry": "876000h"
  }
}

```

### 生成CA证书

```shell
cfssl gencert -initca ca-csr.json | cfssljson -bare ca
```
### 生成etcd证书

```json
cat etcd-csr.json

{
  "CN": "etcd",
  "hosts": [
    "10.206.100.12",
    "10.206.100.15",
    "10.206.100.16",
    "127.0.0.1"
  ],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "HangZhou",
      "L": "XS",
      "O": "k8s",
      "OU": "System"
    }
  ]
}
```

```shell
cfssl gencert -initca ca-csr.json | cfssljson -bare ca

cfssl gencert \
        -ca=ca.pem \
        -ca-key=ca-key.pem \
        -config=ca-config.json \
        -profile=kubernetes etcd-csr.json | cfssljson -bare etcd
```

## 证书分发和二进制分发

```shell
scp -ar ssl root@xxx:/etc/etcd/

scp etcd* root@xxx:/usr/bin/
```

## 创建部署单元

```shell

~]# cat /etc/systemd/system/etcd.service
[Unit]
Description=Etcd Server
After=network.target
After=network-online.target
Wants=network-online.target
Documentation=https://github.com/coreos

[Service]
Type=notify
WorkingDirectory=/var/lib/etcd
ExecStart=/usr/bin/etcd \
  --name=etcd-10.206.100.16 \
  --cert-file=/etc/etcd/ssl/etcd.pem \
  --key-file=/etc/etcd/ssl/etcd-key.pem \
  --peer-cert-file=/etc/etcd/ssl/etcd.pem \
  --peer-key-file=/etc/etcd/ssl/etcd-key.pem \
  --trusted-ca-file=/etc/etcd/ssl/ca.pem \
  --peer-trusted-ca-file=/etc/etcd/ssl/ca.pem \
  --initial-advertise-peer-urls=https://10.206.100.16:2380 \
  --listen-peer-urls=https://10.206.100.16:2380 \
  --listen-client-urls=https://10.206.100.16:2379,http://127.0.0.1:2379 \
  --advertise-client-urls=https://10.206.100.16:2379 \
  --initial-cluster-token=etcd-cluster-0 \
  --initial-cluster=etcd-10.206.100.12=https://10.206.100.12:2380,etcd-10.206.100.15=https://10.206.100.15:2380,etcd-10.206.100.16=https://10.206.100.16:2380 \
  --initial-cluster-state=new \
  --data-dir=/var/lib/etcd \
  --wal-dir=/var/lib/etcd \
  --snapshot-count=50000 \
  --auto-compaction-retention=1 \
  --auto-compaction-mode=periodic \
  --max-request-bytes=10485760 \
  --quota-backend-bytes=8589934592
Restart=always
RestartSec=15
LimitNOFILE=65536
OOMScoreAdjust=-999

[Install]
WantedBy=multi-user.target
```

- 完整参数列表请使用 `etcd --help` 查询
- 注意etcd 即需要服务器证书也需要客户端证书，为方便使用一个peer 证书代替两个证书
- `--initial-cluster-state` 值为 `new` 时，`--name` 的参数值必须位于 `--initial-cluster` 列表中
- `--snapshot-count` `--auto-compaction-retention` 一些性能优化参数，请查阅etcd项目文档
- 设置`--data-dir` 和`--wal-dir` 使用不同磁盘目录，可以避免磁盘io竞争，提高性能，具体请参考etcd项目文档
## 启动服务
```
systemctl daemon-reload && systemctl restart etcd
```

## 验证etcd集群状态

- systemctl status etcd 查看服务状态
- journalctl -u etcd 查看运行日志
- 在任一 etcd 集群节点上执行如下命令

```shell
# 根据hosts中配置设置shell变量 $NODE_IPS
export NODE_IPS="192.168. 192.168. 192.168."
for ip in ${NODE_IPS}; do
  ETCDCTL_API=3 etcdctl \
  --endpoints=https://${ip}:2379  \
  --cacert=/etc/etcd/ssl/ca.pem \
  --cert=/etc/etcd/ssl/etcd.pem \
  --key=/etc/etcd/ssl/etcd-key.pem \
  endpoint health; done

for ip in ${NODE_IPS}; do
  ETCDCTL_API=3 etcdctl \
  --endpoints=https://${ip}:2379  \
  --cacert=/etc/etcd/ssl/ca.pem \
  --cert=/etc/etcd/ssl/etcd.pem \
  --key=/etc/etcd/ssl/etcd-key.pem \
  --write-out=table endpoint status; done
```

预期结果：

```
https://192.168:2379 is healthy: successfully committed proposal: took = 2.210885ms
https://192.168:2379 is healthy: successfully committed proposal: took = 2.784043ms
https://192.168:2379 is healthy: successfully committed proposal: took = 3.275709ms
```

三台 etcd 的输出均为 healthy 时表示集群服务正常。