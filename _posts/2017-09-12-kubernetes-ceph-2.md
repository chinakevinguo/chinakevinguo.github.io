---
layout: post
title: kubernetes ceph 笔记 2
categories: [kubernetes,ceph]
description: kubernetes ceph
keywords: kubernetes,ceph
---

> 其实我们在上一章的时候，已经简单的介绍了ceph中的基础组件以及其他软体架构，那么这一章，主要是来详细的介绍下ceph的工作原理、流程以及更实际的操作，比如ceph 内部的CRUSH bucket调整，PG/PGP参数调整等。并在最后简单的介绍了些CEPH集群硬件要求等。

### 寻址流程

通过对Ceph系统中寻址流程的了解来熟悉Ceph中的几个概念，寻址流程如下图

![Addressing](/images/posts/Addressing.png)

**上图的几个概念说明如下：**

#### 1.File

此处的file就是用户需要存储或者访问的文件。对于一个基于Ceph开发的对象存储而言，这个file也就对应于应用中的"对象"，也就是用户直接操作的“对象”。

#### 2.Object

Object是Ceph中最小存储单元，是由一个数据和一个元数据绑定的整体，元数据中存放了具体数据的属性信息等。Object和File区别在于，object的最大size由RADOS限定（通常为2MB或者4MB），以便实现底层存储的组织管理。因此，当上层应用向RADOS存入size很大的File时，需要将File切分成统一大小的一系列Object(最后一个大小可以不同)进行存储。

#### 3.PG

PG的用途就是对Object的存储进行组织和位置映射。具体而言，一个PG负责组织若干个object(可以为数千个甚至更多)，但一个object只能被映射到一个PG中，即PG和object之间是“一对多”映射关系。同时，一个PG会被映射到n个OSD上，而每个OSD上都会承载大量的PG，即PG和OSD之间是“多对多”映射关系。在实践中，n至少为2，如果用于生产环境，则至少为3。一个OSD上的PG则可达到数百个。事实上，PG数量的设置牵扯到数据分布的均匀性问题。这一点后面再讲。

#### 4.OSD

这个[上一篇](https://kevinguo.me/2017/09/06/kubernetes-ceph-1/)已经讲过了，这里就不再赘述。唯一需要说明的是，OSD的数量事实上也关系到系统的数据分布均匀性，因此其数量不应太少。在实践中，至少也得数十上百个的量级才有助于Ceph系统的设计发挥其应有的优势。

#### 5.CRUSH

在传统的文件存储系统中，元数据占着及其重要的位置，每次系统中数据更新时，元数据会首先被更新，然后才是实际数据的写入；在较小的存储系统中(GB/TB)，这种将元数据存储在某个固定几点或者磁盘阵列中的做法还能满足需求；但当数据量增大到PB/ZB时，元数据查找的性能会成为很大的一个瓶颈；而且元数据的统一存放还可能造成单点故障，即当元数据丢失后，实际数据将无法被找回；与传统文件存储系统不同的是，Ceph使用`CRUSH`算法来精确的计算数据应该被写入那里/从哪里读取；CRUSH按需计算元数据，而不是存储元数据，从而解决了传统文件存储系统的瓶颈。


#### 6.CRUSH计算

CRUSH是伪随机算法，算法通过每个设备的权重(容量大小)来计算数据对象的分布。对象分布式由cluster map和data distribution policy决定的。
cluster map由device和bucket组成。描述了可用存储资源和层级结构(比如有多少个机架，每个机架上有多少个服务器，每个服务器上有多少个磁盘)。
data distribution policy由 placement rules组成。rule决定了每个数据对象有多少个副本，这些副本存储的限制条件(比如3个副本放在不同的机架中)。

在Ceph中，元数据的计算和负载也是分布式的，并且只有在需要的时候才会执行，元数据的计算过程称之为CRUSH计算，不同于其他分布式文件系统，Ceph的CRUSH计算是由客户端使用自己的资源来完成，从而去除了中心查找带来的性能及单节点故障问题。

CRUSH查找时候，ceph client先通过MON 获取cluster map，然后获取到通过Hash算法算出的PG ID，最后再根据CRUSH算法计算出PG中，主和次OSD的ID，最终找到OSD的位置，进行数据的读写。


#### 7.层级的Cluster map

在Ceph中，CRUSH是可以支持各种基础设施和用户自定义的；CRUSH设备列表中预先定义了一系列的设备，用户可以通过自定义配置将不同的OSD分配到不同的区域。这样就在物理上避免了所有数据都在同一个机架上，防止某个机架突然塌了以后数据全部丢失。

#### 8.恢复和再平衡

在Ceph集群中，如果OSD挂了，且老师处于`degraded`状态，Ceph 都会将其标记为 down 和 out 状态；然后默认情况下 Ceph 会等待 300秒之后进行数据恢复和再平衡，这个值可以通过在配置文件中的 mon osd down out interval 参数来调整


#### 总结

基于上述的定义，便可以对寻址流程进行解释了。具体而言，Ceph中的寻址至少要经历三次映射：

1.File -> Object

将用户操作的File，映射为RADOS能处理的object，每个object都会有一个唯一的oid。本质上就是按照object的最大size对file进行切分，切分成大小一致的object(最后的大小可以不一样)

2.Object -> PG

当File被映射为一个或多个object后，就需要将object映射到PG中,得到PG ID。这有一个计算公式

hash(oid) & mask --> pgid

由此可见，其计算由两步组成。首先是使用Ceph指定的静态hash算法计算出oid的hash值。然后将这个随机值和mask(mask的值=PG数-1)按位相与，最终得到PG ID。

3.PG -> OSD

最后根据前面得到的PG ID，通过CRUSH算法最终找到OSD的位置。

### Ceph组件调整及操作

1.pool 操作

```bash
# 列出池
ceph osd lspools

# 在配置文件中调整默认PG 数量以及副本数
osd pool default size = 5
osd pool default pg num = 100
osd pool default pgp num = 100

# 创建池
ceph osd pool create k8s-pool 30

# 获取存储池选项值
ceph osd pool get k8s-pool pg_num/pgp_num

# 调整副本数
ceph osd pool set k8s-pool size 10

# 获取对象副本数
ceph osd dump | grep 'replicated size'

# 删除池
ceph osd pool delete k8s-pool k8s-pool --yes-i-really-really-mean-it
```

2.object操作

```bash
# 将对象放入到池内
rados put test-object testfile.txt -p cephfs_data

# 列出池中对象
rados ls -p cephfs_data

# 检查池中对象位置
ceph osd map cephfs_data test-object

# 删除对象
rados rm test-object -p cephfs_data
```

3.PG和PGP操作

预设Ceph集群中的PG数至关重要，公式如下:

```bash
PG 总数 = (OSD 数 * 100) / 最大副本数
```

集群中单个池的PG数计算公式如下：

```bash
PG 总数 = (OSD 数 * 100) / 最大副本数 / 池数
```

PGP是为了实现定位而设计的PG，PGP的值应该和PG数量保持一致；pgp_num 数值才是 CRUSH 算法采用的真实值。虽然 pg_num 的增加引起了PG的分割，但是只有当 pgp_num增加以后，数据才会被迁移到新PG中，这样才会重新开始平衡。

获取PG和PGP的方式如下：

```bash
ceph osd pool get cephfs_data pg_num
ceph osd pool get cephfs_data pgp_num
```

调整方式如下：

```bash
ceph osd pool set cephfs_data pgp_num 32
ceph osd pool set cephfs_data pgp_num 32
```

3.Cluster map 操作

```bash
# 查看现有集群布局
# 基本上一台机器上一个osd
$  ceph osd tree
ID WEIGHT  TYPE NAME             UP/DOWN REWEIGHT PRIMARY-AFFINITY
-1 0.28134 root default                                            
-2 0.04689     host k8s-master01                                   
 0 0.04689         osd.0              up  1.00000          1.00000
-3 0.04689     host k8s-master02                                   
 1 0.04689         osd.1              up  1.00000          1.00000
-4 0.04689     host k8s-master03                                   
 2 0.04689         osd.2              up  1.00000          1.00000
-5 0.04689     host k8s-node01                                     
 3 0.04689         osd.3              up  1.00000          1.00000
-6 0.04689     host k8s-node02                                     
 4 0.04689         osd.4              up  1.00000          1.00000
-7 0.04689     host k8s-registry                                   
 5 0.04689         osd.5              up  1.00000          1.00000


# 添加逻辑上的机架
ceph osd crush add-bucket rack01 rack
ceph osd crush add-bucket rack02 rack
ceph osd crush add-bucket rack03 rack

# 将机器移动到不同的机架上
ceph osd crush move k8s-master01 rack=rack01
ceph osd crush move k8s-master02 rack=rack02
ceph osd crush move k8s-master03 rack=rack03

# 移动每个机架到默认的根下
ceph osd crush move rack01 root=default
ceph osd crush move rack02 root=default
ceph osd crush move rack03 root=default
```

最终集群整体布局如下，我们可以看到每个机器都被分配到了对应的机架下面，从逻辑上进行了分隔
```bash
$ ceph osd tree
ID  WEIGHT  TYPE NAME                 UP/DOWN REWEIGHT PRIMARY-AFFINITY
 -1 0.28134 root default                                                
 -5 0.04689     host k8s-node01                                         
  3 0.04689         osd.3                  up  1.00000          1.00000
 -6 0.04689     host k8s-node02                                         
  4 0.04689         osd.4                  up  1.00000          1.00000
 -7 0.04689     host k8s-registry                                       
  5 0.04689         osd.5                  up  1.00000          1.00000
 -8 0.04689     rack rack01                                             
 -2 0.04689         host k8s-master01                                   
  0 0.04689             osd.0              up  1.00000          1.00000
 -9 0.04689     rack rack02                                             
 -3 0.04689         host k8s-master02                                   
  1 0.04689             osd.1              up  1.00000          1.00000
-10 0.04689     rack rack03                                             
 -4 0.04689         host k8s-master03                                   
  2 0.04689             osd.2              up  1.00000          1.00000

```
