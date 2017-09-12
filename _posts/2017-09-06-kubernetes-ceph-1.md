---
layout: post
title: kubernetes ceph 笔记 1
categories: [kubernetes,ceph]
description: kubernetes ceph
keywords: kubernetes,ceph
---

> 该教程主要是为statefulset有状态服务集群提供持久化存储提供基础，在讲statefulset之前，我们先搭建我们的ceph集群
> ；具备了极好的可靠性、统一性；经过近几年的发展，ceph开辟了一个全新的数据存储途径。ceph具备了企业级存储的分布式、可大规模扩展、没有当节点故障等特点，越来越受人们的青睐。

### 简介

Ceph是一个符合POSIX、开源的分布式存储系统，不论你是想提供`ceph 对象存储`或`ceph 块设备`，还是想部署一个`ceph文件系统`或者把ceph作为他用，所有 `Ceph存储集群` 的部署都始于部署一个个 `ceph节点`、`网络`和`ceph存储集群`，ceph 存储集群至少需要一个 `ceph monitor`和两个`osd守护进程`。而运行`ceph文件系统客户端`，则必须需要MDS(元数据服务器)。

基础组件:

* Ceph OSDs: Ceph OSD 守护进程的功能是存储数据，处理数据的复制、恢复、回填、再均衡，并通过检查其他OSD守护进程的心跳来向 Ceph Monitors 提供一些监控信息。当Ceph 存储集群设定为有2个副本时，至少需要2个OSD守护进程，集群才能到到`active+clean`状态(Ceph 默认有3个副本，你可以调整`osd poll default size`)

* Ceph Monitors: Ceph Monitor基于PAXOS算法维护着 展示集群状态的各种图表，包括监视器图、OSD图、PG图、CRUSH图。Ceph 保存着发生在Monitors、OSD和PG上的每一次状态变更的历史信息(称为 epoch)

* MDSs：Ceph MDS为 Ceph 文件系统存储元数据(也就是说，Ceph块设备和Ceph对象存储是不是用MDS)。MDS使得POSIX文件系统的用户们，可以在部队Ceph存储集群造成负担的前提下，执行诸如 `ls`、`find`等基本命令


下图展示了Ceph的基础架构图

![ceph-topu](/images/posts/Ceph-soft-topu.png)

#### 1.基础存储系统RADOS

RADOS (Reliable Autonomic Distributed Object Store),这一层本身就是一个完整的对象存储系统，包括Ceph的基础服务(OSD,MON,MDS)，所有存储在ceph中的用户数据实际上最终都是由这一层来存储的。而Ceph的高可用，高扩展性，高自动化等特性本质上也是有这一层所提供的，因此，RADOS是ceph的核心精华部分。

#### 2.基础库LibRados

这一层是对RADOS进行抽象和封装，并向上层提供不同的API，这样上层的RBD、RGW、CephFS才能访问RADOS，RADOS所提供的原生librados API包括C和C++两种。

#### 3.高层存储应用接口

这一层包含了RGW、RBD和CephFS这几个部分，其作用是在librados库的基础上提供抽象层次更高，更便于应用和用户端使用的上层接口。

* RGW: 是一个提供与S3和Swift兼容的RESTful API的gateway，以供对应的对象存储应用开发使用。通过RGW可以将RADOS响应转化为HTTP响应，同样也可以将外部的HTTP响应状花为RADOS调用。

* RBD：提供一个标准的块设备接口服务。

* CephFS: 提供一个POSIX兼容的分布式文件系统。

#### 4.client层

这一层其实就是不同场景下对于ceph各个应用接口的各种应用方式，例如基于librados直接开发的对象存储应用，基于RGW开发的对象存储应用，基于RBD实现的云硬盘等等。

#### 5.其他

一个Cluster逻辑上可以划分为多个Pool，一个Pool由若干个逻辑PG组成。

一个文件会被切分为多个Object，每个Object会被映射到一个PG，每个PG 会根据CRUSH算法映射到一组OSD，其中第一个OSD（Primary OSD）为主，其余是备，OSD间通过心跳来互相监控存活状态。

CRUSH： CRUSH是ceph使用的数据分布算法，类似一致性哈希，让数据分配到预期的地方。

### 一.快速安装

#### 1.1 安装前准备

所需机器如下

| IP | HostName | OS | role |
| :---: | :---: | :---: | :---: |
| 172.30.33.31 | deploy-node | centos7.3.1611 | deploy node |
| 172.30.33.90 | k8s-master01 | centos7.3.1611 | monitor osd node1 |
| 172.30.33.91 | k8s-master02 | centos7.3.1611 | osd node2 |
| 172.30.33.92 | k8s-master03 | centos7.3.1611 | osd node3 |

##### 在管理节点上操作

1.执行如下命令

```bash
$ sudo yum install -y yum-utils && sudo yum-config-manager --add-repo https://dl.fedoraproject.org/pub/epel/7/x86_64/ && sudo yum install --nogpgcheck -y epel-release && sudo rpm --import /etc/pki/rpm-gpg/RPM-GPG-KEY-EPEL-7 && sudo rm /etc/yum.repos.d/dl.fedoraproject.org*
```

2.将软件包源加入软件仓库

```bash
$ vim /etc/yum.repos.d/ceph.repo

[ceph]
name=ceph
baseurl=http://mirrors.163.com/ceph/rpm-jewel/el7/x86_64/
gpgcheck=0
[ceph-noarch]
name=cephnoarch
baseurl=http://mirrors.163.com/ceph/rpm-jewel/el7/noarch/
gpgcheck=0
```

3.更新并安装`ceph-deploy`

```bash
$ sudo yum update && sudo yum install ceph-deploy
```

4.配置从部署机器到所有其他节点的免密钥登录，具体参考[这里](https://kevinguo.me/2017/07/06/ansible-client/)

##### 在节点上操作

1.安装epel源

```bash
$ yum install yum-plugin-priorities
$ yum install epel-release -y
```

2.校对时间，由于ceph使用Paxos算法保证数据一致性，所以安装前要先保证各个节点的时间同步

```bash
$ sudo yum install ntp ntpdate ntp-doc

$ ntpdate 0.cn.pool.ntp.org
```

3.开放所需端口或关闭防火墙

```bash
$ system stop firewalld
$ sudo firewall-cmd --zone=public --add-port=6789/tcp --permanent
```

4.关闭selinux

```bash
$ sudo setenforce 0
```

#### 1.2 创建集群

1.由于ceph-deploy工具部署集群前需要创建一些集群配置信息，其保存在`ceph.conf`文件中，这个文件将来会被复制到每个节点的 `/etc/ceph/ceph.conf`

```bash
# 创建集群配置目录
mkdir ceph-cluster && cd ceph-cluster
# 创建 monitor-node
ceph-deploy new k8s-master01
# 追加 OSD 副本数量(测试虚拟机总共有3台)
echo "osd pool default size = 3" >> ceph.conf
```

2.创建集群使用 `ceph-deploy`工具在部署节点上执行即可

```bash
# 安装ceph
$ ceph-deploy install k8s-master01 k8s-master02 k8s-master03
```
**注意：在部署节点部署的时候，可能会因为网络原因导致无法安装ceph和ceph-radosgw，这时候，我们在各个节点上手动安装一下**
```bash
# 添加ceph 163源
[ceph]
name=ceph
baseurl=http://mirrors.163.com/ceph/rpm-jewel/el7/x86_64/
gpgcheck=0
[ceph-noarch]
name=cephnoarch
baseurl=http://mirrors.163.com/ceph/rpm-jewel/el7/noarch/
gpgcheck=0

$ yum install ceph ceph-radosgw -y
```

3.初始化monitor node 和密钥文件

```bash
$ ceph-deploy mon create-initial
```

4.在osd节点上创建一个目录作为 osd 存储，并修改其权限,千万别创建在`/usr/local`目录下，否则，你的osd可能会无法挂载

```bash
$ ssh k8s-master01
$ sudo mkdir /data/osd1
$ chown -R ceph:ceph osd1
$ exit

$ ssh k8s-master2
$ sudo mkdir /data/osd2
$ chown -R ceph:ceph osd2
$ exit

$ ssh k8s-master3
$ sudo mkdir /data/osd3
$ chown -R ceph:ceph osd3
$ exit
```


5.然后，在管理节点上初始化 osd

```bash
$ ceph-deploy osd prepare k8s-master01:/data/osd1 k8s-master02:/data/osd2 k8s-master03:/data/osd3
```

6.最后，在管理节点上激活 osd

```bash
$ ceph-deploy osd activate k8s-master01:/data/osd1 k8s-master02:/data/osd2 k8s-master03:/data/osd3
```

7.在管理节点上部署 ceph cli 工具和密钥文件

```bash
$ ceph-deploy admin k8s-master01 k8s-master02 k8s-master03
```

8.确保你对 `ceph.client.admin.keyring`有正确的操作权限，在每个节点上执行

```bash
$ sudo chmod +r /etc/ceph/ceph.client.admin.keyring
```

9.最后，检查集群的健康状态

```bash
$ ceph health
HEALTH_OK

$ ceph osd tree
ID WEIGHT  TYPE NAME             UP/DOWN REWEIGHT PRIMARY-AFFINITY
-1 0.14067 root default                                            
-2 0.04689     host k8s-master01                                   
 0 0.04689         osd.0              up  1.00000          1.00000
-3 0.04689     host k8s-master02                                   
 1 0.04689         osd.1              up  1.00000          1.00000
-4 0.04689     host k8s-master03                                   
 2 0.04689         osd.2              up  1.00000          1.00000
```

我们看到状态是OK的，而且每个节点上的osd的状态都是up的


如果在某些地方碰到麻烦，想从头再来，可以用下列命令来清楚配置：

```bash
$ ceph-deploy purgedata {ceph-node} [{ceph-node}]
$ ceph-deploy forgetkeys
```

用下列命令可以连Ceph安装包一起清除：

```bash
$ ceph-deploy purge {ceph-node} [{ceph-node}]
```

### 二.操作集群

#### 2.1 基础操作

```bash
# 创建MDS
$ ceph-deploy mds create {ceph-node}

# 创建RGW
$ ceph-deploy rgw create {ceph-node}

# 添加mon
$ echo "public network = 172.30.33.0/24" >> ceph.conf
$ ceph-deploy mon add {ceph-node}

# 查看仲裁
$ ceph quorum_status --format json-pretty
```

#### 2.2 对象存储测试

```bash
# 创建pool
# rados mkpool {pool-name}
$ rados mkpool test-pool

# 创建测试文件
$ dd if=/dev/zero of=testfile bs=1G count=1

# 创建一个对象(这时候也将对象放入了pool中)
# rados put {object-name} {file-path} --pool={pool-name}
$ rados put test-object testfile --pool=test-pool

# 检查存储池，确认ceph存储了此对象
$ rados -p test-pool ls

# 定位对象，会输出对象位置
$ ceph osd map test-pool test-object
osdmap e42 pool 'test-pool' (3) object 'test-file' -> pg 3.b79653d4 (3.4) -> up ([1,5,2,3,4], p1) acting ([1,5,2,3,4], p1)

# 删除对象
$ rados -p data rm test-object

# 删除存储池
$ rados rmpool test-pool test-pool --yes-i-really-really-mean-it
```
随着集群的运行，对象的位置可能会动态改变。Ceph有动态均衡机制，无需手动干预即可完成。

#### 2.3 块存储测试

**官方建议RBD和OSD最好不要在同一台物理机上(除非它们都是VM)**

1.确认你使用了合适的内核版本，详情[参见](http://docs.ceph.com/docs/master/start/os-recommendations/)

```bash
lab_release -a
uname -a
```

2.在管理节点上用 `ceph-deploy`安装ceph

```bash
ceph-deploy install {rbd-client}
```

3.在管理节点上部署ceph cli工具和密钥

```bash
ceph-deploy admin {rbd-client}
```

4.在rbd节点上创建块设备image

```bash
rbd create test-block --size 4096
```

5.映射image到块设备

```bash
rbd map test-block --name client.admin
```

**在上面的map映射操作时，会出现如下报错**

```bash
$ rbd map test-block --name client.admin
rbd: sysfs write failed
RBD image feature set mismatch. You can disable features unsupported by the kernel with "rbd feature disable".
In some cases useful info is found in syslog - try "dmesg | tail" or so.
rbd: map failed: (6) No such device or address
```

**大致意思是说features不匹配，可以通过disable features关掉一些特性来让内核支持。这是因为在Ceph高本本进行 map image时，默认ceph在创建image(上文test-block)时，会增加很多features，这些features需要内核支持，centos7上的支持有限，所以，我们需要关掉一些**

我们可以用 `rbd info data` 看看创建的image目前有哪些features

```bash
$ rbd info test-block
rbd image 'test-block':
	size 4096 MB in 1024 objects
	order 22 (4096 kB objects)
	block_name_prefix: rbd_data.10bb238e1f29
	format: 2
	features: layering, exclusive-lock, object-map, fast-diff, deep-flatten
	flags:
```

在features中，我们可以看到默认开启了很多：

* layering 支持分层
* exclusive-lock 支持独占锁
* object-map 支持对象映射(依赖exclusive-lock)
* fast-diff 快速计算差异(依赖object-map)
* deep-flatten 支持快照扁平化操作

**而实际上在CentOS7的3.10内核中只支持layering，所以我们需要手动关闭一些features，然后重新map；如果想要一劳永逸，可以在 ceph.conf 中加入 rbd_default_features = 1 来设置默认 features(数值仅是 layering 对应的 bit 码所对应的整数值)**

6.关闭不支持的特性之后重新map

```bash
# 关闭不支持的features
$ rbd feature disable test-block exclusive-lock, object-map, fast-diff, deep-flatten

# 重新map
$ rbd map test-block --name client.admin
/dev/rbd0
```

7.格式化之后挂载到系统目录

```bash
# 格式化
$ mkfs.xfs /dev/rbd0
meta-data=/dev/rbd0              isize=512    agcount=9, agsize=130048 blks
         =                       sectsz=512   attr=2, projid32bit=1
         =                       crc=1        finobt=0, sparse=0
data     =                       bsize=4096   blocks=1048576, imaxpct=25
         =                       sunit=1024   swidth=1024 blks
naming   =version 2              bsize=4096   ascii-ci=0 ftype=1
log      =internal log           bsize=4096   blocks=2560, version=2
         =                       sectsz=512   sunit=8 blks, lazy-count=1
realtime =none                   extsz=4096   blocks=0, rtextents=0

# 挂载
$ mkdir test-block
$ mount /dev/rbd0 test-block

# 写入测试
$ dd if=/dev/zero of=test-block/test-file bs=1G count=1
1+0 records in
1+0 records out
1073741824 bytes (1.1 GB) copied, 2.96071 s, 363 MB/s

$ ls test-block/
test-file
```

#### 2.4 CephFS 测试

1.创建MDS

```bash
$ ceph-deploy mds create k8s-node01 k8s-node02 k8s-registry
```

2.创建pool和fs，创建pool需要指定PG数量

```bash
ceph osd pool create cephfs_data 32
ceph osd pool create cephfs_metadata 32
ceph fs new test-fs cephfs_metadata cephfs_data
```

**PG 概念：**

> 当Ceph集群接受到存储请求时，ceph会将一个文件会切分为多个Object，每个Object会被映射到一个PG，每个PG 会根据CRUSH算法映射到一组OSD(根据副本数)；一般来说增加PG的数量能降低OSD负载，一般每个OSD大约分配50～100PG，关于PG数量指定，一般遵循以下公式
> * 集群PG总数 = (OSD总数 * 100)/数据最大副本数
> * 单个存储池PG数 = (OSD总数 * 100)/数据最大副本数/存储池数

**注意：PG的最终结果应当以最接近以上计算公式的2的N次幂(向上取值)；如我的虚拟机环境的每个存储池 PG数 = 6(OSD) * 100 / 5(副本数) / 4（4个存储池）= 30，向上取2的N次幂为32(即，2的5次方=32，最接近30)**

3.挂载CephFS有两种方式，一种是使用内核驱动挂载，一种是使用 `ceph-fuse`用户空间挂载

内核挂载需要提取ceph管理key，方式如下：

在密钥文件中找到与某用户对于的密钥

```bash
$ cat /etc/ceph/ceph.client.admin.keyring
[client.admin]
	key = AQBferZZEKXFLxAAhlzElpm2MhhbBGB4TnNVkA==
```

复制密钥到文件中保存，并确保其权限

```bash
echo "AQBferZZEKXFLxAAhlzElpm2MhhbBGB4TnNVkA==" > ceph-key
```

创建目录挂载

```bash
mkdir test-fs
mount -t ceph 172.30.33.90:6789:/ /root/test-fs -o name=admin,secretfile=ceph-key

#写入数据测试
$ dd if=/dev/zero of=test-fs/test-fs bs=1G count=1
1+0 records in
1+0 records out
1073741824 bytes (1.1 GB) copied, 2.77355 s, 387 MB/s
```

`ceph-fuse`用户空间挂载的方式也比较简单，需要先安装ceph-fuse，同时也需要key

```bash
# 按照前面的步骤添加ceph源
$ vi /etc/yum.repos.d/ceph.repo
[ceph]
name=ceph
baseurl=http://mirrors.163.com/ceph/rpm-jewel/el7/x86_64/
gpgcheck=0
[ceph-noarch]
name=cephnoarch
baseurl=http://mirrors.163.com/ceph/rpm-jewel/el7/noarch/
gpgcheck=0

# 安装ceph-fuse
yum install ceph-fuse -y

# 复制配置和ceph key到client端
sudo mkdir -p /etc/ceph
sudo scp root@172.30.33.91:/etc/ceph/ceph.conf /etc/ceph/ceph.conf
sudo scp root@172.30.33.91:/etc/ceph/ceph.client.admin.keyring /etc/ceph/ceph.client.admin.keyring

# 创建目录挂载
mkdir test-fs-fuse
$ sudo ceph-fuse -m 172.30.33.91:6789 test-fs-fuse
ceph-fuse[60551]: starting ceph client
2017-09-12 14:47:24.137929 7f63d9719ec0 -1 init, newargv = 0x7f63e51d4840 newargc=11
ceph-fuse[60551]: starting fuse

# 写入数据测试
$ sudo dd if=/dev/zero of=test-fs-fuse/test-fs-fuse bs=1G count=1
1+0 records in
1+0 records out
1073741824 bytes (1.1 GB) copied, 10.5426 s, 102 MB/s

# 查看确认，发现我们上面通过内核挂载的文件也还在
$ ls -lh
total 2.0G
-rw-r--r-- 1 root root 1.0G Sep 12 13:59 test-fs
-rw-r--r-- 1 root root 1.0G Sep 12 14:49 test-fs-fuse
```

#### 2.5 Ceph对象网关

1.对象网关创建

```bash
# 创建RGW
$ ceph-deploy rgw create k8s-node02
```

2.直接访问`http://ceph-node-ip:7480`返回结果如下

```xml
<ListAllMyBucketsResult xmlns="http://s3.amazonaws.com/doc/2006-03-01/">
<Owner>
<ID>anonymous</ID>
<DisplayName/>
</Owner>
<Buckets/>
</ListAllMyBucketsResult>
```

这就说明网关OK了，但是因为没有读写环境，所以暂时测不了。

本文参考了[ceph官方文档](http://docs.ceph.com/docs/master/start/)及漠然的[ceph笔记(一)](https://mritd.me/2017/05/27/ceph-note-1/)部分
