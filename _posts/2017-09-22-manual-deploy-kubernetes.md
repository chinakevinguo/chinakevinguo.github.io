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

> 在安装之前，我们要确认，我们具体需要准备哪些系统组件

* docker
* [etcd-3.2.7](https://github.com/coreos/etcd/releases/download/v3.2.7/etcd-v3.2.7-linux-amd64.tar.gz)(etcd、etcdctl)
* [kubernetes-server-1.7.6](https://dl.k8s.io/v1.7.6/kubernetes-server-linux-amd64.tar.gz)(kube-apiserver、kube-controller-manager、kube-scheduler)
* [kubernetes-node-1.7.6](https://dl.k8s.io/v1.7.6/kubernetes-node-linux-amd64.tar.gz)(kube-proxy、kubelet、kubectl)

##### 1.3 自签名证书

> 因为所有的组件之间都是通过证书认证的方式来进行通信的，所以我们还得确认下，我们到底需要哪些证书

* CA
* etcd
* kube-apiserver
* kube-controller-manager
* kube-scheduler
* kube-proxy
* kubelet
* kube-admin

##### 1.4 系统配置

> 关闭所有节点的selinux、iptables、firewalld

```bash
systemctl stop iptables
systemctl stop firewalld
systemctl disable iptables
systemctl disable firewalld

vi /etc/selinux/config
SELINUX=disable
```

> 在所有节点上创建 `/etc/sysctl.d/k8s.conf`文件，添加如下内容

```bash
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1

# 执行命令使其生效
sysctl -p /etc/sysctl.d/k8s.conf
```

> 在所有节点上编辑 `/etc/hosts`文件，配置host通信

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

> 所有证书分发，二进制文件分发，配置文件分发，都将在 `k8s-console` 上执行，所以该节点主机对集群内所有节点设置了免密钥登录

> 具体过程可参考[免密钥登录](https://kevinguo.me/2017/07/06/ansible-client/)

### 二.创建验证

> 因为所有组件和apiserver进行通信，都需要使用证书来进行认证，所以这里我们使用CloudFlare的PKI工具集 cfssl 来生成CA证书和其密钥文件

> **如果你的kube-controller-manager、kube-scheduler同apiserver之间通信不需要进行证书认证(毕竟他们都在同一台机器上)，那么下面有关kube-controller-manager、kube-scheduler的证书步骤可以忽略；而在该实验中，我考虑到后面假若它们不在同一台机器上，所以也记录了kube-controller-manager、kube-scheduler的证书创建配置过程**

##### 2.1 安装cfssl

```bash
wget https://pkg.cfssl.org/R1.2/cfssl_linux-amd64

wget https://pkg.cfssl.org/R1.2/cfssljson_linux-amd64

chmod +x cfssl_linux-amd64 cfssljson_linux-amd64

mv cfssl_linux-amd64 /usr/local/bin/cfssl
mv cfssljson_linux-amd64 /usr/local/bin/cfssljson
```

##### 2.2 创建CA证书配置，生成CA证书和私钥

> 先用 `cfssl` 命令生成包含默认配置的 `config.json`和 `csr.json`文件

```bash
mkdir /opt/ssl
cd /opt/ssl

cfssl print-defaults config > config.json
cfssl print-defaults csr > csr.json
```

> 然后分别修改这两个文件为如下内容

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

> 生成CA 证书和私钥

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

> 在`/opt/ssl` 下添加文件 `etcd-csr.json`，内容如下

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

> 生成etcd证书和密钥

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

> 在`/opt/ssl` 下添加文件 `kube-apiserver-csr.json`，内容如下

**kube-apiserver-csr.json**

```json
{
  "CN": "kubernetes",
  "hosts": [
    "127.0.0.1",
    "172.29.151.1",
    "172.29.151.2",
    "172.29.151.3",
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

> 生成kube-apiserver证书和私钥

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

> 在`/opt/ssl` 下添加文件 `kube-controller-manager-csr.json`，内容如下

**kube-controller-manager-csr.json**

```json
{
  "CN": "system:kube-controller-manager",
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
      "O": "system:kube-controller-manager",
      "OU": "System"
    }
  ]
}
```

> 生成kube-controller-manager证书和私钥

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

> 在`/opt/ssl` 下添加文件 `kube-scheduler-csr.json`，内容如下

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

> 生成kube-scheduler证书和私钥

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

> 在`/opt/ssl` 下添加文件 `kube-admin-csr.json`，内容如下

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

> 生成kube-admin证书和私钥

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

> 在`/opt/ssl` 下添加文件 `kube-proxy-csr.json`，内容如下

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

> 生成kube-proxy证书和私钥

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

> kubelet 其实也可以手动通过CA来进行签发，但是这只能针对少数机器，毕竟我们在进行证书签发的时候，是需要绑定对应Node的IP的，如果node太多了，加IP就会很幸苦， 所以这里我们使用TLS 认证，由apiserver自动给符合条件的node签发证书，允许节点加入集群。

> kubelet 首次启动时想kube-apiserver发送TLS Bootstrapping请求，kube-apiserver验证kubelet请求中的token是否与它配置的token一致，如果一致则自动为kubelet生成证书和密钥。具体参考[kubelet-tls-bootstrapping](https://k8smeetup.github.io/docs/admin/kubelet-tls-bootstrapping/)

> **我们在k8s-console上生成token并分发到所有的master节点**

```bash
cd /opt/ssl

head -c 16 /dev/urandom | od -An -t x | tr -d ' '
04d9b6c6fd3ed8a3488b3b0913e87d64

vim token.csv
04d9b6c6fd3ed8a3488b3b0913e87d64,kubelet-bootstrap,10001,"system:kubelet-bootstrap"
```

##### 2.11 证书分发

> 既然证书都创建好了，那么这时候，我们就需要将对应的证书分发到对应的节点上去

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

##### 2.11 token分发

```bash
cd /opt/ssl

for IP in `seq 1 3`;do
  ssh root@172.29.151.$IP mkdir -p /etc/kubernetes/known_token
  scp token.csv root@172.29.151.$IP:/etc/kubernetes/known_token/
  ssh root@172.29.151.$IP chown -R kube:kube /etc/kubernetes/known_token
done
```
### 三.分发二进制文件

##### 分发etcd

```bash
tar zxvf etcd-v3.2.7-linux-amd64.tar.gz
cd etcd-v3.2.7-linux-amd64

for IP in `seq 1 3`;do
  scp etcd etcdctl root@172.29.151.$IP:/usr/bin/
done
```

##### 分发kubernetes master

```bash
tar zxvf kubernetes-server-linux-amd64.tar.gz
cd kubernetes/server/bin

for IP in `seq 1 3`;do
  scp kube-apiserver kube-controller-manager kube-scheduler root@172.29.151.$IP:/usr/local/bin/
done
```

##### 分发kubernetes node

```bash
tar zxvf kubernetes-node-linux-amd64.tar.gz
cd kubernetes/node/bin

for IP in `seq 5 7`;do
  scp kubelet kube-proxy root@172.29.151.$IP:/usr/local/bin/
done
```

##### 分发kubectl,etcdctl

```bash
cd kubernetes/node/bin
cp kubectl /usr/local/bin/

cd etcd-v3.2.7-linux-amd64
cp etcdctl /usr/bin/
```


### 四.etcd集群部署

##### 添加etcd为系统服务

> 在每个master节点上添加etcd启动文件/usr/lib/systemd/system/etcd.service

```bash
# etcd1

[Unit]
Description=etcd server
After=network.target
After=network-online.target
Wants=network-online.target

[Service]
Type=notify
User=etcd
WorkingDirectory=/var/lib/etcd/
EnvironmentFile=-/etc/etcd/etcd.conf
ExecStart=/usr/bin/etcd \
  --name etcd1 \
  --cert-file=/etc/etcd/ssl/etcd.pem \
  --key-file=/etc/etcd/ssl/etcd-key.pem \
  --peer-cert-file=/etc/etcd/ssl/etcd.pem \
  --peer-key-file=/etc/etcd/ssl/etcd-key.pem \
  --trusted-ca-file=/etc/etcd/ssl/ca.pem \
  --peer-trusted-ca-file=/etc/etcd/ssl/ca.pem \
  --initial-advertise-peer-urls https://172.29.151.1:2380 \
  --listen-peer-urls https://172.29.151.1:2380 \
  --listen-client-urls https://172.29.151.1:2379,https://127.0.0.1:2379 \
  --advertise-client-urls https://172.29.151.1:2379 \
  --initial-cluster-token k8s_etcd \
  --initial-cluster etcd1=https://172.29.151.1:2380,etcd2=https://172.29.151.2:2380,etcd3=https://172.29.151.3:2380 \
  --initial-cluster-state new \
  --data-dir=/var/lib/etcd
Restart=on-failure
RestartSec=5
```


```bash
# etcd2

[Unit]
Description=etcd server
After=network.target
After=network-online.target
Wants=network-online.target

[Service]
Type=notify
User=etcd
WorkingDirectory=/var/lib/etcd/
EnvironmentFile=-/etc/etcd/etcd.conf
ExecStart=/usr/bin/etcd \
  --name etcd2 \
  --cert-file=/etc/etcd/ssl/etcd.pem \
  --key-file=/etc/etcd/ssl/etcd-key.pem \
  --peer-cert-file=/etc/etcd/ssl/etcd.pem \
  --peer-key-file=/etc/etcd/ssl/etcd-key.pem \
  --trusted-ca-file=/etc/etcd/ssl/ca.pem \
  --peer-trusted-ca-file=/etc/etcd/ssl/ca.pem \
  --initial-advertise-peer-urls https://172.29.151.2:2380 \
  --listen-peer-urls https://172.29.151.2:2380 \
  --listen-client-urls https://172.29.151.2:2379,https://127.0.0.1:2379 \
  --advertise-client-urls https://172.29.151.2:2379 \
  --initial-cluster-token k8s_etcd \
  --initial-cluster etcd1=https://172.29.151.1:2380,etcd2=https://172.29.151.2:2380,etcd3=https://172.29.151.3:2380 \
  --initial-cluster-state new \
  --data-dir=/var/lib/etcd
Restart=on-failure
RestartSec=5
```


```bash
# etcd3

[Unit]
Description=etcd server
After=network.target
After=network-online.target
Wants=network-online.target

[Service]
Type=notify
User=etcd
WorkingDirectory=/var/lib/etcd/
EnvironmentFile=-/etc/etcd/etcd.conf
ExecStart=/usr/bin/etcd \
  --name etcd3 \
  --cert-file=/etc/etcd/ssl/etcd.pem \
  --key-file=/etc/etcd/ssl/etcd-key.pem \
  --peer-cert-file=/etc/etcd/ssl/etcd.pem \
  --peer-key-file=/etc/etcd/ssl/etcd-key.pem \
  --trusted-ca-file=/etc/etcd/ssl/ca.pem \
  --peer-trusted-ca-file=/etc/etcd/ssl/ca.pem \
  --initial-advertise-peer-urls https://172.29.151.3:2380 \
  --listen-peer-urls https://172.29.151.3:2380 \
  --listen-client-urls https://172.29.151.3:2379,https://127.0.0.1:2379 \
  --advertise-client-urls https://172.29.151.3:2379 \
  --initial-cluster-token k8s_etcd \
  --initial-cluster etcd1=https://172.29.151.1:2380,etcd2=https://172.29.151.2:2380,etcd3=https://172.29.151.3:2380 \
  --initial-cluster-state new \
  --data-dir=/var/lib/etcd
Restart=on-failure
RestartSec=5
```

##### 验证etcd 集群状态

> 查看etcd集群状态

```bash
etcdctl \
  --endpoints=https://172.29.151.1:2379 \
  --cert-file=/etc/etcd/ssl/etcd.pem \
  --ca-file=/etc/etcd/ssl/ca.pem \
  --key-file=/etc/etcd/ssl/etcd-key.pem \
  cluster-health

member 31a0c451ae46a2d0 is healthy: got healthy result from https://172.29.151.1:2379
member 72b18fe792c0a463 is healthy: got healthy result from https://172.29.151.3:2379
member d0c073403f6edbd3 is healthy: got healthy result from https://172.29.151.2:2379
cluster is healthy
```

> 查看etcd 集群成员

```bash
etcdctl \
  --endpoints=https://172.29.151.1:2379 \
  --cert-file=/etc/etcd/ssl/etcd.pem \
  --ca-file=/etc/etcd/ssl/ca.pem \
  --key-file=/etc/etcd/ssl/etcd-key.pem \
  member list

31a0c451ae46a2d0: name=etcd1 peerURLs=https://172.29.151.1:2380 clientURLs=https://172.29.151.1:2379 isLeader=true
72b18fe792c0a463: name=etcd3 peerURLs=https://172.29.151.3:2380 clientURLs=https://172.29.151.3:2379 isLeader=false
d0c073403f6edbd3: name=etcd2 peerURLs=https://172.29.151.2:2380 clientURLs=https://172.29.151.2:2379 isLeader=false
```

### 五.kube-apiserver部署

##### 添加kube-apiserver为系统服务

> 在每个master节点上添加/usr/lib/systemd/system/kube-apiserver.service，注意修改为各自节点的ip地址

```bash
# 创建日志目录文件
mkdir -p /var/log/kubernetes
```

> kube-apiserver.service文件内容如下

**注意：安全端口监听在172.29.151.1，提供给node节点访问，非安全端口监听在127.0.0.1，只提供给同一台机器上的kube-controller-manager和kube-scheduler访问，这样就保证了安全性和稳定性**

```bash
[Unit]
Description=Kubernetes API Server
After=network.target
After=etcd.service

[Service]
EnvironmentFile=-/etc/kubernetes/apiserver
ExecStart=/usr/local/bin/kube-apiserver \
  --admission-control=NamespaceLifecycle,LimitRanger,ServiceAccount,PersistentVolumeLabel,DefaultStorageClass,ResourceQuota,DefaultTolerationSeconds \
  --advertise-address=172.29.151.1 \
  --bind-address=172.29.151.1 \
  --insecure-bind-address=127.0.0.1 \
  --service-cluster-ip-range=10.254.0.0/16 \
  --service-node-port-range=30000-32000 \
  --allow-privileged=true \
  --apiserver-count=3 \
  --logtostderr=true \
  --v=0 \
  --audit-log-maxage=30 \
  --audit-log-maxbackup=3 \
  --audit-log-maxsize=100 \
  --audit-log-path=/var/log/kubernetes/audit.log \
  --authorization-mode=RBAC \
  --enable-swagger-ui=true \
  --event-ttl=1h \
  --secure-port=6443 \
  --insecure-port=8080 \
  --etcd-servers=https://172.29.151.1:2379,https://172.29.151.2:2379,https://172.29.151.3:2379 \
  --etcd-cafile=/etc/etcd/ssl/ca.pem \
  --etcd-certfile=/etc/etcd/ssl/etcd.pem \
  --etcd-keyfile=/etc/etcd/ssl/etcd-key.pem \
  --storage-backend=etcd3 \
  --tls-cert-file=/etc/kubernetes/ssl/kube-apiserver.pem \
  --tls-private-key-file=/etc/kubernetes/ssl/kube-apiserver-key.pem \
  --client-ca-file=/etc/kubernetes/ssl/ca.pem \
  --service-account-key-file=/etc/kubernetes/ssl/kube-apiserver-key.pem \
  --token-auth-file=/etc/kubernetes/known_token/token.csv \
  --experimental-bootstrap-token-auth=true \
  --kubelet-https=true \
  --anonymous-auth=False
Restart=on-failure
Type=notify
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
```

##### 启动kube-apiserver

```bash
systemctl daemon-reload
systemctl enable kube-apiserver
systemctl start kube-apiserver
systemctl status kube-apiserver
```

### 六.kube-controller-manager部署

##### 添加kube-controller-manager为系统服务

> 在每个master节点上添加/usr/lib/systemd/system/kube-controller-manager.service

```bash
[Unit]
Description=Kubernetes Controller Manager
After=network.target
After=kube-apiserver.service

[Service]
ExecStart=/usr/local/bin/kube-controller-manager \
  --address=127.0.0.1 \
  --master=http://127.0.0.1:8080 \
  --allocate-node-cidrs=true \
  --service-cluster-ip-range=10.254.0.0/16 \
  --cluster-cidr=10.233.0.0/16 \
  --cluster-name=kubernetes \
  --cluster-signing-cert-file=/etc/kubernetes/ssl/ca.pem \
  --cluster-signing-key-file=/etc/kubernetes/ssl/ca-key.pem \
  --service-account-private-key-file=/etc/kubernetes/ssl/kube-apiserver-key.pem \
  --root-ca-file=/etc/kubernetes/ssl/ca.pem \
  --leader-elect=true \
  --node-monitor-grace-period=40s \
  --node-monitor-period=5s \
  --pod-eviction-timeout=5m0s \
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

##### 启动kube-controller-manager

```bash
systemctl daemon-reload
systemctl enable kube-controller-manager
systemctl start kube-controller-manager
systemctl status kube-controller-manager
```


### 七.kube-scheduler部署

##### 添加kube-scheduler为系统服务

> 在每个master节点上添加/usr/lib/systemd/system/kube-scheduler.service

```bash
[Unit]
Description=kube-scheduler
After=network.target
After=kube-apiserver.service

[Service]
EnvironmentFile=-/etc/kubernetes/scheduler
ExecStart=/usr/local/bin/kube-scheduler \
      --address=127.0.0.1 \
	    --logtostderr=true \
	    --v=2 \
	    --master=127.0.0.1:8080 \
	    --leader-elect=true
Restart=on-failure
Type=simple
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
```

##### 启动kube-scheduler

```bash
systemctl daemon-reload
systemctl enable kube-scheduler
systemctl start kube-scheduler
systemctl status kube-scheduler
```

### 八.Master HA配置

> 目前所谓的kubernetes HA 其实主要是API Server的HA，master上的其他组件，比如kube-controller-manager、kube-scheduler都是通过etcd做选举。而API Server一般有两种方式做HA；一种是多个API Server 做聚合为 VIP，另一种使用nginx反向代理，这里我们采用nginx的方式，如下图

![apiserver-ha](/images/posts/apiserver-ha.png)

**kube-controller-manager、kube-scheduler通过etcd选举，而且与master直接通过127.0.0.1:8080通信，而其他node，则需要在每个node上启动一个nginx，每个nginx反代所有apiserver，node上的kubelet、kube-proxy、kubectl连接本地nginx代理端口，当nginx发现无法连接后端时会自动踢掉出问题的apiserver，从而实现api server的HA**

##### 在每个node节点和k8s-console上创建nginx代理

> 在每个节点上新建配置目录

```bash
mkdir -p /etc/nginx
```

> 在配置文件/etc/nginx/nginx.conf中下写入代理配置

```bash
error_log stderr notice;

worker_processes auto;
events {
  multi_accept on;
  use epoll;
  worker_connections 1024;
}

stream {
    upstream kube_apiserver {
        least_conn;
        server 172.29.151.1:6443;
        server 172.29.151.2:6443;
        server 172.29.151.3:6443;
    }

    server {
        listen        0.0.0.0:6443;
        proxy_pass    kube_apiserver;
        proxy_timeout 10m;
        proxy_connect_timeout 1s;
    }
}
```

> 更新权限

```bash
chmod +r /etc/nginx/nginx.conf
```

##### 将nginx配置为docker启动，同时用systemd来进行守护

> 在每个node节点上添加/etc/systemd/system/nginx-proxy.service

```bash
[Unit]
Description=kubernetes apiserver docker wrapper
Wants=docker.socket
After=docker.service

[Service]
User=root
PermissionsStartOnly=true
ExecStart=/usr/bin/docker run -p 127.0.0.1:6443:6443 \
                              -v /etc/nginx:/etc/nginx \
                              --name nginx-proxy \
                              --net=host \
                              --restart=on-failure:5 \
                              --memory=512M \
                              nginx:1.13.3-alpine
ExecStartPre=-/usr/bin/docker rm -f nginx-proxy
ExecStop=/usr/bin/docker stop nginx-proxy
Restart=always
RestartSec=15s
TimeoutStartSec=30s

[Install]
WantedBy=multi-user.target
```


##### 配置开机启动

```bash
systemctl daemon-reload
systemctl start nginx-proxy
systemctl enable nginx-proxy
```

**最后我们在k8s-console上执行kubectl试试**

```bash
$ kubectl --server=https://127.0.0.1:6443 --certificate-authority=/etc/kubernetes/ssl/ca.pem --client-certificate=/etc/kubernetes/ssl/kube-admin.pem --client-key=/etc/kubernetes/ssl/kube-admin-key.pem get cs
NAME                 STATUS    MESSAGE              ERROR
controller-manager   Healthy   ok                   
scheduler            Healthy   ok                   
etcd-1               Healthy   {"health": "true"}   
etcd-0               Healthy   {"health": "true"}   
etcd-2               Healthy   {"health": "true"}   
```

### 九. 配置kubectl访问apiserver

> 前面我们使用kubect打印除了kubernetes核心组件的状态，但是每次使用的时候都需要指定apiserver的地址以及证书之类的，实在是有点繁琐，接下来，我们在k8s-console上创建kubeconfig文件。

```bash
cd /etc/kubernetes
export KUBE_APISERVER="https://127.0.0.1:6443"

# 设置集群参数
kubectl config set-cluster kubernetes \
 --certificate-authority=/opt/ssl/ca.pem \
 --embed-certs=true \
 --server=${KUBE_APISERVER} \
 --kubeconfig=admin.conf

# 设置客户端认证参数
kubectl config set-credentials kubernetes-admin \
  --client-certificate=/opt/ssl/kube-admin.pem \
  --embed-certs=true \
  --client-key=/opt/ssl/kube-admin-key.pem \
  --kubeconfig=admin.conf

# 设置上下文参数
kubectl config set-context kubernetes-admin@kubernetes \
  --cluster=kubernetes \
  --user=kubernetes-admin \
  --kubeconfig=admin.conf

# 设置默认上下文
kubectl config use-context kubernetes-admin@kubernetes --kubeconfig=admin.conf

# cp成~/.kube/config
cp /etc/kubernetes/ssl/admin.conf ~/.kube/config
```

##### 试试看是否生效

```bash
$ kubectl get cs
NAME                 STATUS    MESSAGE              ERROR
scheduler            Healthy   ok                   
controller-manager   Healthy   ok                   
etcd-2               Healthy   {"health": "true"}   
etcd-1               Healthy   {"health": "true"}   
etcd-0               Healthy   {"health": "true"}   
```

### 十.kubelet配置

> kubelet启动时向kube-apiserver发送 TLS bootstrapping请求，需要先将 bootstrap token 文件中的 kubelet-bootstrap用户赋予system:node-bootstrapper角色，然后kubelet才有权限创建认证请求。


##### kubelet角色授权

```bash
# 在k8s-console上执行绑定操作
kubectl create clusterrolebinding kubelet-bootstrap --clusterrole=system:node-bootstrapper --user=kubelet-bootstrap
```


##### 在k8s-console上生成kubelet kubeconfig文件

> 配置集群

```bash
kubectl config set-cluster kubernetes \
  --certificate-authority=/opt/ssl/ca.pem \
  --embed-certs=true \
  --server=https://127.0.0.1:6443 \
  --kubeconfig=bootstrap.kubeconfig
```

> 配置客户端认证参数

```bash
kubectl config set-credentials kubelet-bootstrap \
  --token=04d9b6c6fd3ed8a3488b3b0913e87d64 \
  --kubeconfig=bootstrap.kubeconfig
```

> 配置上下文关联

```bash
kubectl config set-context default \
  --cluster=kubernetes \
  --user=kubelet-bootstrap \
  --kubeconfig=bootstrap.kubeconfig
```

> 配置默认上下文

```bash
kubectl config use-context default --kubeconfig=bootstrap.kubeconfig
```

> 分发bootstrap.kubeconfig文件到每个node节点

```bash
cd /etc/kubernetes
for IP in `seq 5 7`;do
  scp bootstrap.kubeconfig root@172.29.151.$IP:/etc/kubernetes/
done
```

##### 添加kubelet为系统服务

> 创建kubelet工作目录

```bash
mkdir /var/lib/kubelet
```

> 添加/usr/lib/systemd/system/kubelet.service,注意修改你成你自己节点的ip

```bash
[Unit]
Description=Kubernetes Kubelet
After=docker.service
Requires=docker.service

[Service]
WorkingDirectory=/var/lib/kubelet
ExecStart=/usr/local/bin/kubelet \
  --cgroup-driver=systemd \
  --address=172.29.151.5 \
  --hostname-override=172.29.151.5 \
  --pod-infra-container-image=gcr.io/google_containers/pause-amd64:3.0 \
  --experimental-bootstrap-kubeconfig=/etc/kubernetes/bootstrap.kubeconfig \
  --kubeconfig=/etc/kubernetes/kubelet.kubeconfig \
  --require-kubeconfig \
  --cert-dir=/etc/kubernetes/ssl \
  --cluster_dns=10.254.0.2 \
  --cluster_domain=cluster.local. \
  --hairpin-mode promiscuous-bridge \
  --allow-privileged=true \
  --serialize-image-pulls=false \
  --logtostderr=true \
  --max-pods=512 \
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

##### 启动kubelet

```bash
systemctl daemon-reload
systemctl enable kubelet
systemctl start kubelet
systemctl status kubelet
```

##### 签发证书，验证nodes

> 查看csr，我们发现状态为Pending

```bash
$ kubectl get csr
NAME                                                   AGE       REQUESTOR           CONDITION
node-csr-MIqovZHmrYMe1Y6AspcfU6_keLdSWfbUqg4pcK-Hb9w   2m        kubelet-bootstrap   Pending
node-csr-el6foG3yw6_9xCu1vC_upuT-xLR9Z9ASBNj5isBFcsY   2m        kubelet-bootstrap   Pending
node-csr-oPWmprgtrRixLZXUvEFKnHI2qZEGorzHKZ1ktLMdGS8   5m        kubelet-bootstrap   Pending
```

> 签发证书

```bash
$ kubectl certificate approve node-csr-oPWmprgtrRixLZXUvEFKnHI2qZEGorzHKZ1ktLMdGS8 node-csr-el6foG3yw6_9xCu1vC_upuT-xLR9Z9ASBNj5isBFcsY node-csr-MIqovZHmrYMe1Y6AspcfU6_keLdSWfbUqg4pcK-Hb9w
certificatesigningrequest "node-csr-oPWmprgtrRixLZXUvEFKnHI2qZEGorzHKZ1ktLMdGS8" approved
certificatesigningrequest "node-csr-el6foG3yw6_9xCu1vC_upuT-xLR9Z9ASBNj5isBFcsY" approved
certificatesigningrequest "node-csr-MIqovZHmrYMe1Y6AspcfU6_keLdSWfbUqg4pcK-Hb9w" approved
```

> 查看node

```bash
$ kubectl get nodes
NAME           STATUS    AGE       VERSION
172.29.151.5   Ready     3m        v1.7.6
172.29.151.6   Ready     45s       v1.7.6
172.29.151.7   Ready     12s       v1.7.6
```

> 成功后会自动生成配置文件和密钥

```bash
$ ll /etc/kubernetes/ssl
-rw-r--r-- 1 root root 1042 Oct  9 17:46 kubelet-client.crt
-rw------- 1 root root  227 Oct  9 17:18 kubelet-client.key
-rw-r--r-- 1 root root 1111 Oct  9 17:46 kubelet.crt
-rw------- 1 root root 1675 Oct  9 17:46 kubelet.key


$ ll /etc/kubernetes/kubelet.kubeconfig
-rw------- 1 root root 2260 Oct  9 17:46 /etc/kubernetes/kubelet.kubeconfig
```


### 十一.kube-proxy 配置

##### 在k8s-console上生成kube-proxy kubeconfig文件

> 配置集群

```bash
kubectl config set-cluster kubernetes \
  --certificate-authority=/opt/ssl/ca.pem \
  --embed-certs=true \
  --server=https://127.0.0.1:6443 \
  --kubeconfig=kube-proxy.kubeconfig
```

> 配置客户端认证

```bash
kubectl config set-credentials kube-proxy \
  --client-certificate=/opt/ssl/kube-proxy.pem \
  --client-key=/opt/ssl/kube-proxy-key.pem \
  --embed-certs=true \
  --kubeconfig=kube-proxy.kubeconfig
```

> 配置上下文

```bash
kubectl config set-context default \
  --cluster=kubernetes \
  --user=kube-proxy \
  --kubeconfig=kube-proxy.kubeconfig
```

> 配置默认上下文

```bash
kubectl config use-context default --kubeconfig=kube-proxy.kubeconfig
```

> 分发到各个节点的/etc/kubernetes 目录

```bash
for IP in `seq 5 7`;do
  scp kube-proxy.kubeconfig root@172.29.151.$IP:/etc/kubernetes/
done
```

##### 添加kube-proxy为系统服务

> 创建 kube-proxy目录

```bash
mkdir -p /var/lib/kube-proxy
```

> 添加/usr/lib/systemd/system/kube-proxy.service，注意修改为自己的节点ip

```bash
[Unit]
Description=Kubernetes Kube-Proxy Server
After=network.target

[Service]
WorkingDirectory=/var/lib/kube-proxy
ExecStart=/usr/local/bin/kube-proxy \
  --bind-address=172.29.151.5 \
  --hostname-override=172.29.151.5 \
  --cluster-cidr=10.254.0.0/16 \
  --kubeconfig=/etc/kubernetes/kube-proxy.kubeconfig \
  --logtostderr=true \
  --v=2
Restart=on-failure
RestartSec=5
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
```

##### 启动kube-proxy

```bash
systemctl daemon-reload
systemctl enable kube-proxy
systemctl start kube-proxy
systemctl status kube-proxy
```

### 十二.calico配置

> 网络组件采用calico，calico部署比较简单，只需要create 一下yml文件即可，具体参考[calico官方文档](https://docs.projectcalico.org/v2.6/getting-started/kubernetes/installation/) ，在使用calico网络的时候，官方的要求如下

* kubelet 必须配置使用CNI插件`--network-plugin=cni`
* kube-proxy 必须以`iptables`的模式启动
* kube-proxy 不能使用`--masquerade-all`启动(会与calico policy冲突)
* kubernetes networkpolicy api 至少需要kubernetes 1.3 版本以上
* 如果开启了RBAC，那么需要注意需要创建clusterrole和clusterrolebinding

##### 在每个节点上修改kubelet.service

> 修改kubelet配置，增加`--network-plugin=cni`

```bash
vi /usr/lib/systemd/system/kubelet.service

--network-plugin=cni
```

> 重启kubelet

```bash
systemctl daemon-reload
systemctl restart kubelet.service
systemctl status kubelet.service
```

##### 准备依赖包和文件

> 下载calico.yaml 和rbac.yaml

```bash
wget https://docs.projectcalico.org/v2.6/getting-started/kubernetes/installation/hosted/calico.yaml
wget https://docs.projectcalico.org/v2.6/getting-started/kubernetes/installation/rbac.yaml
```

> 下载镜像

```bash
quay.io/calico/node:v2.6.1
quay.io/calico/cni:v1.11.0
quay.io/calico/kube-controllers:v1.0.0
```

##### 修改配置文件

> 修改etcd_endpoints

```bash
etcd_endpoints: "https://172.29.151.1:2379,https://172.29.151.2:2379,https://172.29.151.3:2379"
```

> 修改calico所需的etcd密钥信息

```bash
etcd_ca: "/calico-secrets/etcd-ca"
etcd_cert: "/calico-secrets/etcd-cert"
etcd_key: "/calico-secrets/etcd-key"
```

> 写入etcd-key、etcd-cert、etcd-ca的base64信息，将括号里面命令执行的结果填入即可

```bash
data:
  etcd-key: (cat /opt/ssl/etcd-key.pem | base64 | tr -d '\n')
  etcd-cert: (cat /opt/ssl/etcd.pem | base64 | tr -d '\n')
  etcd-ca: (cat /opt/ssl/ca.pem | base64 | tr -d '\n')
```

> 修改calico的网络段

```bash
- name: CALICO_IPV4POOL_CIDR
      value: "10.233.0.0/16"
```

> 注释掉calico-node 部分，这部分用systemctl来进行管理，因为用官方文档可能会出现无法获取到IP的情况

```bash
# Calico Version v2.6.1
# https://docs.projectcalico.org/v2.6/releases#v2.6.1
# This manifest includes the following component versions:
#   calico/node:v2.6.1
#   calico/cni:v1.11.0
#   calico/kube-controllers:v1.0.0

# This ConfigMap is used to configure a self-hosted Calico installation.
kind: ConfigMap
apiVersion: v1
metadata:
  name: calico-config
  namespace: kube-system
data:
  # Configure this with the location of your etcd cluster.
  etcd_endpoints: "https://172.29.151.1:2379,https://172.29.151.2:2379,https://172.29.151.3:2379"

  # Configure the Calico backend to use.
  calico_backend: "bird"

  # The CNI network configuration to install on each node.
  cni_network_config: |-
    {
        "name": "k8s-pod-network",
        "cniVersion": "0.1.0",
        "type": "calico",
        "etcd_endpoints": "__ETCD_ENDPOINTS__",
        "etcd_key_file": "__ETCD_KEY_FILE__",
        "etcd_cert_file": "__ETCD_CERT_FILE__",
        "etcd_ca_cert_file": "__ETCD_CA_CERT_FILE__",
        "log_level": "info",
        "mtu": 1500,
        "ipam": {
            "type": "calico-ipam"
        },
        "policy": {
            "type": "k8s",
            "k8s_api_root": "https://__KUBERNETES_SERVICE_HOST__:__KUBERNETES_SERVICE_PORT__",
            "k8s_auth_token": "__SERVICEACCOUNT_TOKEN__"
        },
        "kubernetes": {
            "kubeconfig": "__KUBECONFIG_FILEPATH__"
        }
    }

  # If you're using TLS enabled etcd uncomment the following.
  # You must also populate the Secret below with these files.
  etcd_ca: "/calico-secrets/etcd-ca"
  etcd_cert: "/calico-secrets/etcd-cert"
  etcd_key: "/calico-secrets/etcd-key"

---

# The following contains k8s Secrets for use with a TLS enabled etcd cluster.
# For information on populating Secrets, see http://kubernetes.io/docs/user-guide/secrets/
apiVersion: v1
kind: Secret
type: Opaque
metadata:
  name: calico-etcd-secrets
  namespace: kube-system
data:
  # Populate the following files with etcd TLS configuration if desired, but leave blank if
  # not using TLS for etcd.
  # This self-hosted install expects three files with the following names.  The values
  # should be base64 encoded strings of the entire contents of each file.
  etcd-key: 这块自己对 etcd 相关证书做 base64
  etcd-cert: 这块自己对 etcd 相关证书做 base64
  etcd-ca: 这块自己对 etcd 相关证书做 base64

---

# This manifest installs the calico/node container, as well
# as the Calico CNI plugins and network config on
# each master and worker node in a Kubernetes cluster.
kind: DaemonSet
apiVersion: extensions/v1beta1
metadata:
  name: calico-node
  namespace: kube-system
  labels:
    k8s-app: calico-node
spec:
  selector:
    matchLabels:
      k8s-app: calico-node
  template:
    metadata:
      labels:
        k8s-app: calico-node
      annotations:
        scheduler.alpha.kubernetes.io/critical-pod: ''
        scheduler.alpha.kubernetes.io/tolerations: |
          [{"key": "dedicated", "value": "master", "effect": "NoSchedule" },
           {"key":"CriticalAddonsOnly", "operator":"Exists"}]
    spec:
      hostNetwork: true
      serviceAccountName: calico-node
      containers:
        # Runs calico/node container on each Kubernetes node.  This
        # container programs network policy and routes on each
        # host.
        # 从这里开始注释掉calico-node的部分
        # - name: calico-node
        #   image: quay.io/calico/node:v2.6.1
        #   env:
        #     # The location of the Calico etcd cluster.
        #     - name: ETCD_ENDPOINTS
        #       valueFrom:
        #         configMapKeyRef:
        #           name: calico-config
        #           key: etcd_endpoints
        #     # Choose the backend to use.
        #     - name: CALICO_NETWORKING_BACKEND
        #       valueFrom:
        #         configMapKeyRef:
        #           name: calico-config
        #           key: calico_backend
        #     # Cluster type to identify the deployment type
        #     - name: CLUSTER_TYPE
        #       value: "k8s,bgp"
        #     # Disable file logging so `kubectl logs` works.
        #     - name: CALICO_DISABLE_FILE_LOGGING
        #       value: "true"
        #     # Set Felix endpoint to host default action to ACCEPT.
        #     - name: FELIX_DEFAULTENDPOINTTOHOSTACTION
        #       value: "ACCEPT"
        #     # Configure the IP Pool from which Pod IPs will be chosen.
        #     - name: CALICO_IPV4POOL_CIDR
        #       value: "10.233.0.0/16"
        #     - name: CALICO_IPV4POOL_IPIP
        #       value: "always"
        #     # Disable IPv6 on Kubernetes.
        #     - name: FELIX_IPV6SUPPORT
        #       value: "false"
        #     # Set Felix logging to "info"
        #     - name: FELIX_LOGSEVERITYSCREEN
        #       value: "info"
        #     # Set MTU for tunnel device used if ipip is enabled
        #     - name: FELIX_IPINIPMTU
        #       value: "1440"
        #     # Location of the CA certificate for etcd.
        #     - name: ETCD_CA_CERT_FILE
        #       valueFrom:
        #         configMapKeyRef:
        #           name: calico-config
        #           key: etcd_ca
        #     # Location of the client key for etcd.
        #     - name: ETCD_KEY_FILE
        #       valueFrom:
        #         configMapKeyRef:
        #           name: calico-config
        #           key: etcd_key
        #     # Location of the client certificate for etcd.
        #     - name: ETCD_CERT_FILE
        #       valueFrom:
        #         configMapKeyRef:
        #           name: calico-config
        #           key: etcd_cert
        #     # Auto-detect the BGP IP address.
        #     - name: IP
        #       value: ""
        #     - name: FELIX_HEALTHENABLED
        #       value: "true"
        #   securityContext:
        #     privileged: true
        #   resources:
        #     requests:
        #       cpu: 250m
        #   livenessProbe:
        #     httpGet:
        #       path: /liveness
        #       port: 9099
        #     periodSeconds: 10
        #     initialDelaySeconds: 10
        #     failureThreshold: 6
        #   readinessProbe:
        #     httpGet:
        #       path: /readiness
        #       port: 9099
        #     periodSeconds: 10
        #   volumeMounts:
        #     - mountPath: /lib/modules
        #       name: lib-modules
        #       readOnly: true
        #     - mountPath: /var/run/calico
        #       name: var-run-calico
        #       readOnly: false
        #     - mountPath: /calico-secrets
        #       name: etcd-certs
        # This container installs the Calico CNI binaries
        # and CNI network config file on each node.
        - name: install-cni
          image: quay.io/calico/cni:v1.11.0
          command: ["/install-cni.sh"]
          env:
            # The location of the Calico etcd cluster.
            - name: ETCD_ENDPOINTS
              valueFrom:
                configMapKeyRef:
                  name: calico-config
                  key: etcd_endpoints
            # The CNI network config to install on each node.
            - name: CNI_NETWORK_CONFIG
              valueFrom:
                configMapKeyRef:
                  name: calico-config
                  key: cni_network_config
          volumeMounts:
            - mountPath: /host/opt/cni/bin
              name: cni-bin-dir
            - mountPath: /host/etc/cni/net.d
              name: cni-net-dir
            - mountPath: /calico-secrets
              name: etcd-certs
      volumes:
        # Used by calico/node.
        - name: lib-modules
          hostPath:
            path: /lib/modules
        - name: var-run-calico
          hostPath:
            path: /var/run/calico
        # Used to install CNI.
        - name: cni-bin-dir
          hostPath:
            path: /opt/cni/bin
        - name: cni-net-dir
          hostPath:
            path: /etc/cni/net.d
        # Mount in the etcd TLS secrets.
        - name: etcd-certs
          secret:
            secretName: calico-etcd-secrets

---

# This manifest deploys the Calico Kubernetes controllers.
# See https://github.com/projectcalico/kube-controllers
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: calico-kube-controllers
  namespace: kube-system
  labels:
    k8s-app: calico-kube-controllers
  annotations:
    scheduler.alpha.kubernetes.io/critical-pod: ''
    scheduler.alpha.kubernetes.io/tolerations: |
      [{"key": "dedicated", "value": "master", "effect": "NoSchedule" },
       {"key":"CriticalAddonsOnly", "operator":"Exists"}]
spec:
  # The controllers can only have a single active instance.
  replicas: 1
  strategy:
    type: Recreate
  template:
    metadata:
      name: calico-kube-controllers
      namespace: kube-system
      labels:
        k8s-app: calico-kube-controllers
    spec:
      # The controllers must run in the host network namespace so that
      # it isn't governed by policy that would prevent it from working.
      hostNetwork: true
      serviceAccountName: calico-kube-controllers
      containers:
        - name: calico-kube-controllers
          image: quay.io/calico/kube-controllers:v1.0.0
          env:
            # The location of the Calico etcd cluster.
            - name: ETCD_ENDPOINTS
              valueFrom:
                configMapKeyRef:
                  name: calico-config
                  key: etcd_endpoints
            # Location of the CA certificate for etcd.
            - name: ETCD_CA_CERT_FILE
              valueFrom:
                configMapKeyRef:
                  name: calico-config
                  key: etcd_ca
            # Location of the client key for etcd.
            - name: ETCD_KEY_FILE
              valueFrom:
                configMapKeyRef:
                  name: calico-config
                  key: etcd_key
            # Location of the client certificate for etcd.
            - name: ETCD_CERT_FILE
              valueFrom:
                configMapKeyRef:
                  name: calico-config
                  key: etcd_cert
          volumeMounts:
            # Mount in the etcd TLS secrets.
            - mountPath: /calico-secrets
              name: etcd-certs
      volumes:
        # Mount in the etcd TLS secrets.
        - name: etcd-certs
          secret:
            secretName: calico-etcd-secrets

---

# This deployment turns off the old "policy-controller". It should remain at 0 replicas, and then
# be removed entirely once the new kube-controllers deployment has been deployed above.
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: calico-policy-controller
  namespace: kube-system
  labels:
    k8s-app: calico-policy
spec:
  # Turn this deployment off in favor of the kube-controllers deployment above.
  replicas: 0
  strategy:
    type: Recreate
  template:
    metadata:
      name: calico-policy-controller
      namespace: kube-system
      labels:
        k8s-app: calico-policy
    spec:
      hostNetwork: true
      serviceAccountName: calico-kube-controllers
      containers:
        - name: calico-policy-controller
          image: quay.io/calico/kube-controllers:v1.0.0
          env:
            # The location of the Calico etcd cluster.
            - name: ETCD_ENDPOINTS
              valueFrom:
                configMapKeyRef:
                  name: calico-config
                  key: etcd_endpoints

---

apiVersion: v1
kind: ServiceAccount
metadata:
  name: calico-kube-controllers
  namespace: kube-system

---

apiVersion: v1
kind: ServiceAccount
metadata:
  name: calico-node
  namespace: kube-system
```

##### 安装calico

> 在每个node节点上创建calico所需的目录，并分发证书

```bash
cd /opt/ssl
for IP in `seq 5 7`;do
  ssh root@172.29.151.$IP mkdir -p /etc/calico/certs
  scp ca.pem etcd*.pem root@172.29.151.$IP:/etc/calico/certs/
done
```

> 在每个node节点上添加/etc/calico/calico.env文件,注意修改你自己的ip和hostname

```bash
ETCD_ENDPOINTS="https://172.29.151.1:2379,https://172.29.151.2:2379,https://172.29.151.3:2379"
ETCD_CA_CERT_FILE="/etc/calico/certs/ca.pem"
ETCD_CERT_FILE="/etc/calico/certs/etcd.pem"
ETCD_KEY_FILE="/etc/calico/certs/etcd-key.pem"
CALICO_IP="172.29.151.5"
CALICO_IP6=""
CALICO_LIBNETWORK_ENABLED="true"
CALICO_IPV4POOL_CIDR="10.233.0.0/16"
CALICO_IPV4POOL_IPIP="always"
CALICO_HOSTNAME="k8s-mds-node01"
```

> 在每个node节点上添加calico.service为系统服务

```bash
[Unit]
Description=calico-node
After=docker.service
Requires=docker.service

[Service]
EnvironmentFile=/etc/calico/calico.env
ExecStartPre=-/usr/bin/docker rm -f calico-node
ExecStart=/usr/bin/docker run --net=host --privileged \
 --name=calico-node \
 -e ETCD_ENDPOINTS=${ETCD_ENDPOINTS} \
 -e ETCD_CA_CERT_FILE=${ETCD_CA_CERT_FILE} \
 -e ETCD_CERT_FILE=${ETCD_CERT_FILE} \
 -e ETCD_KEY_FILE=${ETCD_KEY_FILE} \
 -e HOSTNAME=${CALICO_HOSTNAME} \
 -e IP=${CALICO_IP} \
 -e IP6=${CALICO_IP6} \
 -e CALICO_NETWORKING_BACKEND=${CALICO_NETWORKING_BACKEND} \
 -e CALICO_LIBNETWORK_ENABLED=${CALICO_LIBNETWORK_ENABLED} \
 -e CALICO_IPV4POOL_CIDR=${CALICO_IPV4POOL_CIDR} \
 -e CALICO_IPV4POOL_IPIP=${CALICO_IPV4POOL_IPIP} \
 -e CALICO_DISABLE_FILE_LOGGING=${CALICO_DISABLE_FILE_LOGGING} \
 -e FELIX_DEFAULTENDPOINTTOHOSTACTION=RETURN \
 -e FELIX_IPV6SUPPORT=false \
 -e FELIX_LOGSEVERITYSCREEN=info \
 -e AS=${CALICO_AS} \
 -v /var/log/calico:/var/log/calico \
 -v /run/docker/plugins:/run/docker/plugins \
 -v /lib/modules:/lib/modules \
 -v /var/run/calico:/var/run/calico \
 -v /var/run/docker.sock:/var/run/docker.sock \
 -v /etc/calico/certs:/etc/calico/certs:ro \
 --memory=500M --cpu-shares=300 \
 quay.io/calico/node:v2.6.1

Restart=always
RestartSec=10s

ExecStop=-/usr/bin/docker stop calico-node

[Install]
WantedBy=multi-user.target
```

> 启动calico-node

```bash
systemctl daemon-reload
systemctl enable calico-node
systemctl start calico-node
systemctl status calico-node
```
> 在k8s-console上执行calico安装

```bash
$ kubectl apply -f calico.yaml
configmap "calico-config" created
secret "calico-etcd-secrets" created
daemonset "calico-node" created
deployment "calico-policy-controller" created
serviceaccount "calico-policy-controller" created
serviceaccount "calico-node" created


$ kubectl apply -f rbac.yaml
clusterrole "calico-policy-controller" created
clusterrolebinding "calico-policy-controller" created
clusterrole "calico-node" created
clusterrolebinding "calico-node" created
```

##### 验证calico

```bash
$ kubectl get pods -n kube-system
NAME                                       READY     STATUS    RESTARTS   AGE
calico-kube-controllers-3994748863-0dpcp   1/1       Running   0          1h
calico-node-74d64                          1/1       Running   0          14h
calico-node-rbrw3                          1/1       Running   0          14h
calico-node-vtcrs                          1/1       Running   0          14h
```

##### 安装calicoctl

> 在k8s-console上下载calicoctl并分发到各个node节点

```bash
wget https://github.com/projectcalico/calicoctl/releases/download/v1.6.1/calicoctl
chmod +x calicoctl

for IP in `seq 5 7`;do
  scp calicoctl root@172.29.151.$IP:/usr/local/bin/
done
```

> 在节点上看看calico的状态

```bash
$ calicoctl node status
Calico process is running.

IPv4 BGP status
+--------------+-------------------+-------+----------+-------------+
| PEER ADDRESS |     PEER TYPE     | STATE |  SINCE   |    INFO     |
+--------------+-------------------+-------+----------+-------------+
| 172.29.151.6 | node-to-node mesh | up    | 12:40:50 | Established |
| 172.29.151.7 | node-to-node mesh | up    | 12:40:50 | Established |
+--------------+-------------------+-------+----------+-------------+

IPv6 BGP status
No IPv6 peers found.

```

##### 测试跨主机通信

> 创建一个nginxdeployment

```yaml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: nginx-dm
spec:
  replicas: 2
  template:
    metadata:
      labels:
        name: nginx
    spec:
      containers:
        - name: nginx
          image: nginx:alpine
          imagePullPolicy: IfNotPresent
          ports:
            - containerPort: 80

---

apiVersion: v1
kind: Service
metadata:
  name: nginx-svc
spec:
  ports:
    - port: 80
      targetPort: 80
      protocol: TCP
  selector:
    name: nginx
```

> 查看创建结果

```bash
$ kubectl get pods -o wide
NAME                        READY     STATUS    RESTARTS   AGE       IP             NODE
nginx-dm-2214564181-bplwr   1/1       Running   0          3m        10.233.136.3   172.29.151.7
nginx-dm-2214564181-qsl5c   1/1       Running   0          3m        10.233.203.2   172.29.151.6

$ kubectl get deployment
NAME       DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
nginx-dm   2         2         2            2           4m

$ kubectl get svc
NAME         CLUSTER-IP       EXTERNAL-IP   PORT(S)   AGE
kubernetes   10.254.0.1       <none>        443/TCP   14h
nginx-svc    10.254.149.124   <none>        80/TCP    4m
```

> 在pod里ping另一个pod

```bash
$ kubectl exec -ti nginx-dm-2214564181-bplwr /bin/sh
/ # ping 10.233.203.2
PING 10.233.203.2 (10.233.203.2): 56 data bytes
64 bytes from 10.233.203.2: seq=0 ttl=62 time=0.592 ms
64 bytes from 10.233.203.2: seq=1 ttl=62 time=0.894 ms
64 bytes from 10.233.203.2: seq=2 ttl=62 time=0.559 ms
```

> 在node节点上curl测试一下

```bash
$ curl 10.254.149.124
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
```


### 十三.部署DNS

##### 部署集群dns
> 获取对应的yaml文件

```bash
wget https://raw.githubusercontent.com/kubernetes/kubernetes/master/cluster/addons/dns/kube-dns.yaml.sed
mv kube-dns.yaml.sed kube-dns.yaml
```

> 修改如下配置

```bash
sed -i 's/$DNS_DOMAIN/cluster.local/gi' kube-dns.yaml
sed -i 's/$DNS_SERVER_IP/10.254.0.2/gi' kube-dns.yaml
```

> 创建

```bash
kubectl create -f kube-dns.yaml
```

> 查看创建结果

```bash
$ kubectl get pods -n kube-system |grep kube-dns
kube-dns-3468831164-2kl0h                  3/3       Running   0          14m
```

> 进入刚刚创建的nginx pod中访问nginx-svc测试

```bash
$ kubectl exec -ti nginx-dm-2214564181-bplwr /bin/sh
/ # curl nginx-svc
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>

```

> 测试外网

```bash
$ kubectl exec -ti nginx-dm-2214564181-bplwr /bin/sh
/ # curl https://baidu.com
<html>
<head><title>302 Found</title></head>
<body bgcolor="white">
<center><h1>302 Found</h1></center>
<hr><center>bfe/1.0.8.18</center>
</body>
</html>
```

##### 部署dns自动扩容

> 下载对应的yaml文件，不需要任何修改

```bash
wget https://raw.githubusercontent.com/kubernetes/kubernetes/master/cluster/addons/dns-horizontal-autoscaler/dns-horizontal-autoscaler.yaml
```

> 在1.7.6中rbac还是beta版本，所以，这里我们要修改文件中的authentication.k8s.io/v1 为 authentication.k8s.io/vibeta1

```bash
sed -i 's/rbac.authorization.k8s.io\/v1/rbac.authorization.k8s.io\/v1beta1/gi' dns-horizontal-autoscaler.yaml
```
> 创建

```bash
kubectl create -f dns-horizontal-autoscaler.yaml
```

> 查看创建结果

```bash
$ kubectl get pods -n kube-system
NAME                                       READY     STATUS    RESTARTS   AGE
calico-kube-controllers-3994748863-0dpcp   1/1       Running   0          7h
calico-node-74d64                          1/1       Running   0          20h
calico-node-rbrw3                          1/1       Running   0          20h
calico-node-vtcrs                          1/1       Running   0          20h
kube-dns-3468831164-2kl0h                  3/3       Running   0          5h
kube-dns-3468831164-zjgzp                  3/3       Running   0          13m
kube-dns-autoscaler-244676396-bpfpw        1/1       Running   0          13m
```

### 十四.kubernetes周边组件配置

##### kubernetes-dashboard配置

> kubernetes基础环境搭建好之后，我们第一步要搭建的就是我们的kubernetes-dashboard

> 准备所需image

```bash
gcr.io/google_containers/kubernetes-dashboard-amd64:v1.7.1
gcr.io/google_containers/kubernetes-dashboard-init-amd64:v1.0.0
```

> 下载所需yaml文件

```bash
wget https://raw.githubusercontent.com/kubernetes/dashboard/master/src/deploy/recommended/kubernetes-dashboard.yaml
```

> 为了方便测试，我们在最后添加NodePort，后期如果有了ingress或traffic,再将其去掉即可

```bash
kind: Service
apiVersion: v1
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard
  namespace: kube-system
spec:
  type: NodePort
  ports:
    - port: 443
      targetPort: 8443
      nodePort: 30001
  selector:
    k8s-app: kubernetes-dashboard
```

> 创建

```bash
$ kubectl create -f kubernetes-dashboard.yaml

secret "kubernetes-dashboard-certs" created
serviceaccount "kubernetes-dashboard" created
role "kubernetes-dashboard-minimal" created
rolebinding "kubernetes-dashboard-minimal" created
deployment "kubernetes-dashboard" created
service "kubernetes-dashboard" created

```

> 查看创建结果

```bash
$ kubectl get pods -n kube-system -o wide
NAME                                       READY     STATUS    RESTARTS   AGE       IP              NODE
calico-kube-controllers-3994748863-0dpcp   1/1       Running   1          1d        172.29.151.6    172.29.151.6
calico-node-74d64                          1/1       Running   1          1d        172.29.151.6    172.29.151.6
calico-node-rbrw3                          1/1       Running   1          1d        172.29.151.5    172.29.151.5
calico-node-vtcrs                          1/1       Running   1          1d        172.29.151.7    172.29.151.7
kube-dns-3468831164-2kl0h                  3/3       Running   3          23h       10.233.136.10   172.29.151.7
kube-dns-3468831164-zjgzp                  3/3       Running   3          18h       10.233.161.7    172.29.151.5
kube-dns-autoscaler-244676396-bpfpw        1/1       Running   1          18h       10.233.136.9    172.29.151.7
kubernetes-dashboard-3625439193-tgtmm      1/1       Running   0          8s        10.233.136.15   172.29.151.7

$ kubectl get svc  -n kube-system
NAME                   CLUSTER-IP      EXTERNAL-IP   PORT(S)         AGE
kube-dns               10.254.0.2      <none>        53/UDP,53/TCP   23h
kubernetes-dashboard   10.254.116.15   <nodes>       443:30001/TCP   1m
```

> 最后我们来访问 https://$NODEIP:30001试试，我们发现新的kubernetes提供了认证，就算是skip进去之后，也看不到啥东西

![](/images/posts/kubernetes-login.png)

> 这里我们使用token认证，那么token来自于哪呢，我们创建一个kubernetes-dashboard-rbac.yaml,内容如下

```bash
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: dashboard-admin
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: default
  namespace: kube-system
```

> 创建之后，我们来获取它的token值

**我们看到这里的serviceaccount是在kube-system的default的，所以我们直接查看kube-system中的default secret就可以了**

```bash
kubectl create -f kubernetes-dashboard-rbac.yaml

$ kubectl describe secret default-token-d9jjg -n kube-system
Name:		default-token-d9jjg
Namespace:	kube-system
Labels:		<none>
Annotations:	kubernetes.io/service-account.name=default
		kubernetes.io/service-account.uid=458abfc9-aef6-11e7-aa7b-00155dfa7a1a

Type:	kubernetes.io/service-account-token

Data
====
ca.crt:		1346 bytes
namespace:	11 bytes
token:		eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJrdWJlLXN5c3RlbSIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VjcmV0Lm5hbWUiOiJkZWZhdWx0LXRva2VuLWQ5ampnIiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZXJ2aWNlLWFjY291bnQubmFtZSI6ImRlZmF1bHQiLCJrdWJlcm5ldGVzLmlvL3NlcnZpY2VhY2NvdW50L3NlcnZpY2UtYWNjb3VudC51aWQiOiI0NThhYmZjOS1hZWY2LTExZTctYWE3Yi0wMDE1NWRmYTdhMWEiLCJzdWIiOiJzeXN0ZW06c2VydmljZWFjY291bnQ6a3ViZS1zeXN0ZW06ZGVmYXVsdCJ9.gRfeCeQSRPOP7yZ94STPZ8GLb77Gx2wAgyVmyATbyoYR7ZMOgqIOMX0lmgZIzCkA1hFnPHcQ863Q9lW_uvkDbHYWA2B2DrRrdkBOYnq_FF2RM09qrwqspS5u3L0w1vgo7S--Rs-mG-yYnMw0EwBtl9rd6Lx7q59sDvWzU47YoQD3HyYZNuIiaIhuZiugvpkJGeKrrsHpd-wh4_rMcTp0GnUKdqSoIpeth2jvudnu34Wv_Jh5q2rhvhMSgb-qEW7JqB5wnDzXLaxkdW7i5PVDZD5RGCQGDwxqr4opfg53JrJQ9ojEjmR7Q0GfgWyKkudwlBm9nPT0VaW4LJkaM37vpQ
```

> 我们输入token登录看看，发现可以看到内容了

![](/images/posts/kubernetes-dashboard-login-token.png)


##### heapster

> kubernetes-dashboard搭建好之后，我们配套的搭建下heapster

> 准备所需镜像

```bash
gcr.io/google_containers/heapster-grafana-amd64:v4.0.2
gcr.io/google_containers/heapster-amd64:v1.3.0
gcr.io/google_containers/heapster-influxdb-amd64:v1.1.1
```

> 下载所需yaml文件

```bash
wget https://github.com/kubernetes/heapster/archive/v1.4.3.tar.gz
```

> 进入heapster-1.4.3/deploy/kube-config/influxdb，修改一下grafana.yaml里面的镜像版本，如果你想要通过NodePort查看下grafana的数据测试一下，可以注释掉service中的 type: NodePort

```bash
gcr.io/google_containers/heapster-grafana-amd64:v4.2.0 -> gcr.io/google_containers/heapster-grafana-amd64:v4.0.2
type: NodePort
```

> 执行构建

```bash
cd heapster-1.4.3/deploy/kube-config/influxdb
kubectl create -f .

cd heapster-1.4.3/deploy/kube-config/rbac
kubectl create -f .
```

> 修改kubernetes-dashboard.yaml 文件，添加如下内容

```bash
  # - --apiserver-host=http://my-address:port
  - --heapster-host=http://heapster.kube-system.svc.cluster.local
```

> 重新构建kubernetes-dashboard

```bash
kubectl create -f kubernetes-dashboard.yaml
```

> 查看构建结果

```bash
$ kubectl get pods -n kube-system
NAME                                       READY     STATUS    RESTARTS   AGE
calico-kube-controllers-3994748863-0dpcp   1/1       Running   1          1d
calico-node-74d64                          1/1       Running   1          1d
calico-node-rbrw3                          1/1       Running   2          1d
calico-node-vtcrs                          1/1       Running   1          1d
heapster-84017538-54dkm                    1/1       Running   0          1h
kube-dns-3468831164-2kl0h                  3/3       Running   3          1d
kube-dns-3468831164-9hsbm                  3/3       Running   0          3h
kube-dns-autoscaler-244676396-bpfpw        1/1       Running   1          22h
kubernetes-dashboard-2923351285-pzgx5      1/1       Running   0          24m
monitoring-grafana-2115417091-lgqsc        1/1       Running   0          1h
monitoring-influxdb-3570645011-dp51l       1/1       Running   0          1h
```

> 登录kubernetes-dashboard，查看是否有数据了

![](/images/posts/kubernetes-dashboard-heapster.png)

> 登录到grafana查看数据

![](/images/posts/grafana.png)


##### ingress 配置
