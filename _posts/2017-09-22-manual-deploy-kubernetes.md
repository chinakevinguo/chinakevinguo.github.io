---
layout: post
title: 手动搭建kubernetes HA集群
categories: [kubernetes, docker]
description: 手动搭建kubernetes HA集群
keywords: kubernetes,docker
---

> 原有的环境需要迁移，现在需要重新搭建一套kubernetes，而且原来一直是用kargo来搭建，所有组件都是基于docker容器的，感觉有点不稳妥，所以正好这个时候有机会，可以纯手动部署一下，所有的关键组件都以二进制形式部署，并添加为系统服务，这里记录一下。

> 该文档参考了诸多大神的文档,[漠然](https://mritd.me/2017/07/21/set-up-kubernetes-ha-cluster-by-binary/)、[青蛙小白](http://blog.frognew.com/2017/04/install-ha-kubernetes-1.6-cluster.html)，谨请原谅

### 一.环境准备

##### 1.1 系统环境

| IP | HostName | 节点 | OS |
| :--- | :--- | :--- | :--- |
| 172.29.151.1 | k8s-mon-master01 | master MON etcd | centOS7.4.1708 |
| 172.29.151.2 | k8s-mon-master02 | master MON etcd | centOS7.4.1708 |
| 172.29.151.3 | k8s-mon-master03 | master MON etcd | centOS7.4.1708 |
| 172.29.151.4 | k8s-harbor | harbor | centOS7.4.1708 |
| 172.29.151.5 | k8s-mds-node01 | node MDS | centOS7.4.1708 |
| 172.29.151.6 | k8s-mds-node02 | node MDS | centOS7.4.1708 |
| 172.29.151.7 | k8s-mds-node03 | node MDS | centOS7.4.1708 |
| 172.29.151.8 | k8s-console | console | centOS7.4.1708 |

##### 1.2 系统组件

在安装之前，我们要确认，我们具体需要准备哪些系统组件

* docker
* [etcd-3.2.7](https://github.com/coreos/etcd/releases/download/v3.2.7/etcd-v3.2.7-linux-amd64.tar.gz)(etcd、etcdctl)
* [kubernetes-server-1.7.6](https://dl.k8s.io/v1.7.6/kubernetes-server-linux-amd64.tar.gz)(kube-apiserver、kube-controller-manager、kube-scheduler)
* [kubernetes-node-1.7.6](https://dl.k8s.io/v1.7.6/kubernetes-node-linux-amd64.tar.gz)(kube-proxy、kubelet、kubectl)

##### 1.3 自签名证书

因为所有的组件之间都是通过证书认证的方式来进行通信的，所以我们还得确认下，我们到底需要哪些证书

* CA
* etcd
* kube-apiserver
* kube-controller-manager
* kube-scheduler
* kube-proxy
* kubelet
* kube-admin

##### 1.4 系统配置

关闭所有节点的selinux

```bash
vi /etc/selinux/config
SELINUX=disable
```

在所有节点上创建 `/etc/sysctl.d/k8s.conf`文件，添加如下内容

```bash
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1

# 执行命令使其生效
sysctl -p /etc/sysctl.d/k8s.conf
```

在所有节点上编辑 `/etc/hosts`文件，配置host通信

```bash
vi /etc/hosts

172.29.151.1 k8s-mon-master01
172.29.151.2 k8s-mon-master02
172.29.151.3 k8s-mon-master03
172.29.151.4 k8s-harbor
172.29.151.5 k8s-mds-node01
172.29.151.6 k8s-mds-node02
172.29.151.7 k8s-mds-node03
172.29.151.8 k8s-console
```

##### 1.5 创建基本用户

```bash
# 在master节点上创建etcd用户
useradd etcd -d /var/lib/etcd -c "Etcd user" -r -s /sbin/nologin

# 在maaster节点和node节点上创建kube用户
useradd kube  -M -c "Kubernetes user" -r -s /sbin/nologin
```

##### 1.6 在console上配置免密钥登录

所有证书分发，二进制文件分发，配置文件分发，都将在 `k8s-console` 上执行，所以该节点主机对集群内所有节点设置了免密钥登录

具体过程可参考[免密钥登录](https://kevinguo.me/2017/07/06/ansible-client/)

### 二.创建验证

> 因为所有组件和apiserver进行通信，都需要使用证书来进行认证，所以这里我们使用CloudFlare的PKI工具集 cfssl 来生成CA证书和其密钥文件

**如果你的kube-controller-manager、kube-scheduler同apiserver之间通信不需要进行证书认证(毕竟他们都在同一台机器上)，那么下面有关kube-controller-manager、kube-scheduler的证书步骤可以忽略；而在该实验中，我考虑到后面假若它们不在同一台机器上，所以也记录了kube-controller-manager、kube-scheduler的证书创建配置过程**

##### 2.1 安装cfssl

```bash
wget https://pkg.cfssl.org/R1.2/cfssl_linux-amd64

wget https://pkg.cfssl.org/R1.2/cfssljson_linux-amd64

chmod +x cfssl_linux-amd64 cfssljson_linux-amd64

mv cfssl_linux-amd64 /usr/local/bin/cfssl
mv cfssljson_linux-amd64 /usr/local/bin/cfssljson
```

##### 2.2 创建CA证书配置，生成CA证书和私钥

先用 `cfssl` 命令生成包含默认配置的 `config.json`和 `csr.json`文件

```bash
mkdir /opt/ssl
cd /opt/ssl

cfssl print-defaults config > config.json
cfssl print-defaults csr > csr.json
```

然后分别修改这两个文件为如下内容

**config.json**

```json
{
  "signing": {
    "default": {
      "expiry": "87600h"
    },
    "profiles": {
      "kubernetes": {
        "usages": [
            "signing",
            "key encipherment",
            "server auth",
            "client auth"
        ],
        "expiry": "87600h"
      }
    }
  }
}
```

**csr.json**

```json
{
  "CN": "kubernetes",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "Wuhan",
      "L": "Hubei",
      "O": "k8s",
      "OU": "System"
    }
  ]
}
```

生成CA 证书和私钥

```bash
cd /opt/ssl

cfssl gencert -initca csr.json | cfssljson -bare ca

# CA有关证书列表如下
[root@k8s-console ssl]# tree
.
├── ca.csr
├── ca-key.pem
├── ca.pem
├── config.json
└── csr.json
```

##### 2.4 创建etcd证书配置，生成 etcd 证书和私钥

在`/opt/ssl` 下添加文件 `etcd-csr.json`，内容如下

**etcd-csr.json**

```json
{
  "CN": "etcd",
  "hosts": [
    "127.0.0.1",
    "172.29.151.1",
    "172.29.151.2",
    "172.29.151.3"
  ],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "Wuhan",
      "L": "Hubei",
      "O": "k8s",
      "OU": "System"
    }
  ]
}
```

生成etcd证书和密钥

```bash
cd /opt/ssl

cfssl gencert -ca=/opt/ssl/ca.pem \
-ca-key=/opt/ssl/ca-key.pem \
-config=/opt/ssl/config.json \
-profile=kubernetes etcd-csr.json | cfssljson -bare etcd

# etcd 有关证书证书列表如下
ls etcd*
etcd.csr  etcd-csr.json  etcd-key.pem  etcd.pem
```

##### 2.5 创建kube-apiserver证书配置，生成kube-apiserver证书和私钥

在`/opt/ssl` 下添加文件 `kube-apiserver-csr.json`，内容如下

**kube-apiserver-csr.json**

```json
{
  "CN": "kubernetes",
  "hosts": [
    "127.0.0.1",
    "172.29.151.1",
    "172.29.151.2",
    "172.20.151.3",
    "10.254.0.1",
    "localhost",
    "kubernetes",
    "kubernetes.default",
    "kubernetes.default.svc",
    "kubernetes.default.svc.cluster",
    "kubernetes.default.svc.cluster.local"
  ],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "Wuhan",
      "L": "Hubei",
      "O": "k8s",
      "OU": "System"
    }
  ]
}
```

生成kube-apiserver证书和私钥

```bash
cd /opt/ssl

cfssl gencert -ca=/opt/ssl/ca.pem \
-ca-key=/opt/ssl/ca-key.pem \
-config=/opt/ssl/config.json \
-profile=kubernetes kube-apiserver-csr.json | cfssljson -bare kube-apiserver

# 列出kube-apiserver有关证书
ls kube-apiserver*
kube-apiserver.csr  kube-apiserver-csr.json  kube-apiserver-key.pem  kube-apiserver.pem
```

##### 2.6 创建kube-controller-manager证书配置，生成kube-controller-manager证书和私钥

在`/opt/ssl` 下添加文件 `kube-controller-manager-csr.json`，内容如下

**kube-controller-manager-csr.json**

```json
{
  "CN": "system:kube-controller-manager",
  "hosts": [
    "172.29.151.1",
    "172.29.151.2",
    "172.20.151.3"
  ],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "Wuhan",
      "L": "Hubei",
      "O": "system:kube-controller-manager",
      "OU": "System"
    }
  ]
}
```

生成kube-controller-manager证书和私钥

```bash
cd /opt/ssl

cfssl gencert -ca=/opt/ssl/ca.pem \
-ca-key=/opt/ssl/ca-key.pem \
-config=/opt/ssl/config.json \
-profile=kubernetes kube-controller-manager-csr.json | cfssljson -bare kube-controller-manager

# 列出kube-controller-manager有关证书
ls kube-controller-manager*
kube-controller-manager.csr  kube-controller-manager-csr.json  kube-controller-manager-key.pem  kube-controller-manager.pem

```


##### 2.7 创建kube-scheduler证书配置，生成kube-scheduler证书和私钥

在`/opt/ssl` 下添加文件 `kube-scheduler-csr.json`，内容如下

**kube-scheduler-csr.json**

```json
{
  "CN": "system:kube-scheduler",
  "hosts": [
    "172.29.151.1",
    "172.29.151.2",
    "172.29.151.3"
  ],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "Wuhan",
      "L": "Hubei",
      "O": "system:kube-scheduler",
      "OU": "System"
    }
  ]
}
```

生成kube-scheduler证书和私钥

```bash
cd /opt/ssl

cfssl gencert -ca=/opt/ssl/ca.pem \
-ca-key=/opt/ssl/ca-key.pem \
-config=/opt/ssl/config.json \
-profile=kubernetes kube-scheduler-csr.json | cfssljson -bare kube-scheduler

# 列出kube-scheduler有关证书
ls kube-scheduler*
kube-scheduler.csr  kube-scheduler-csr.json  kube-scheduler-key.pem  kube-scheduler.pem
```

##### 2.8 创建kube-admin证书配置，生成kube-admin证书和私钥

在`/opt/ssl` 下添加文件 `kube-admin-csr.json`，内容如下

**kube-admin-csr.json**

```json
{
  "CN": "kube-admin",
  "hosts": [
    "172.29.151.8"
  ],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "Wuhan",
      "L": "Hubei",
      "O": "system:masters",
      "OU": "System"
    }
  ]
}
```

生成kube-admin证书和私钥

```bash
cd /opt/ssl

cfssl gencert -ca=/opt/ssl/ca.pem \
-ca-key=/opt/ssl/ca-key.pem \
-config=/opt/ssl/config.json \
-profile=kubernetes kube-admin-csr.json | cfssljson -bare kube-admin

# 列出kube-admin有关证书
ls kube-admin*
kube-admin.csr  kube-admin-csr.json  kube-admin-key.pem  kube-admin.pem
```


##### 2.9 创建kube-proxy证书配置，生成kube-proxy证书和私钥

在`/opt/ssl` 下添加文件 `kube-proxy-csr.json`，内容如下

**kube-proxy-csr.json**

```json
{
  "CN": "system:kube-proxy",
  "hosts": [
  ],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "Wuhan",
      "L": "Hubei",
      "O": "system:kube-proxy",
      "OU": "System"
    }
  ]
}
```

生成kube-proxy证书和私钥

```bash
cd /opt/ssl

cfssl gencert -ca=/opt/ssl/ca.pem \
-ca-key=/opt/ssl/ca-key.pem \
-config=/opt/ssl/config.json \
-profile=kubernetes kube-proxy-csr.json | cfssljson -bare kube-proxy

# 列出kube-proxy有关证书
ls kube-proxy*
kube-proxy.csr  kube-proxy-csr.json  kube-proxy-key.pem  kube-proxy.pem
```

##### 2.10 kubelet 证书和私钥

kubelet 其实也可以手动通过CA来进行签发，但是这只能针对少数机器，毕竟我们在进行证书签发的时候，是需要绑定对应Node的IP的，如果node太多了，加IP就会很幸苦， 所以这里我们使用TLS 认证，由apiserver自动给符合条件的node签发证书，允许节点加入集群，具体过程，在安装kubelet的时候再说。

##### 2.11 证书分发

既然证书都创建好了，那么这时候，我们就需要将对应的证书分发到对应的节点上去

**master**

```bash
cd /opt/ssl

for IP in `seq 1 3`;do
  ssh root@172.29.151.$IP mkdir -p /etc/kubernetes/ssl
  ssh root@172.29.151.$IP mkdir -p /etc/etcd/ssl
  scp ca*.pem kube-apiserver*.pem kube-controller-manager*.pem kube-scheduler*.pem root@172.29.151.$IP:/etc/kubernetes/ssl/
  ssh root@172.29.151.$IP chown -R kube:kube /etc/kubernetes/ssl
  scp ca*.pem etcd*.pem root@172.29.151.$IP:/etc/etcd/ssl
  ssh root@172.29.151.$IP chown -R etcd:etcd /etc/etcd/ssl
done
```

**node**

```bash
cd /opt/ssl

for IP in `seq 5 7`;do
  ssh root@172.29.151.$IP mkdir -p /etc/kubernetes/ssl
  scp ca*.pem kube-proxy*.pem root@172.29.151.$IP:/etc/kubernetes/ssl/
  ssh root@172.29.151.$IP chown -R kube:kube /etc/kubernetes/ssl
done
```

**console**

```bash
cd /opt/ssl

cp kube-admin*.pem /etc/kubernetes/ssl/
chown -R kube:kube /etc/kubernetes/ssl/
```
