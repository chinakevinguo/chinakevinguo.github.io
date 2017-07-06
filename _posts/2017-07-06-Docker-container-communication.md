---
layout: post
title: Docker基础-docker容器之间通信
categories: [docker]
description: 在默认bridge网桥上容器之间如何通信
keywords: docker
---
本章节主要讲了容器之间是如何在默认网桥上进行通信的，这个`bridge`网络是在docker安装的时候，系统默认创建的。
> **Note:** docker允许创建除了默认`bridge`网络之外的bridge网络
<!--more-->
# 和外网通信
容器能否和外网通信是由两个因素决定的。第一个因素是宿主机是否转发容器的IP包。第二个因素是宿主机的iptables是否允许这个特定的连接。
IP包转发是由系统参数`ip_forward`来进行管理的。如果此参数是1，则只允许在容器之间传递数据包，此时容器是不能访问外网的。通常，你只需要保持docker服务器的默认设置(`ip-forward=true`)即可,并且当服务器启动时，docker会将`ip_forward`设置为1，并且系统内核已经启用，那么这时候哪怕你设置`ip-forward=false`也不会起作用。
检查内核上的设置或手动打开：
```bash
$ sysctl net.ipv4.conf.all.forwarding

net.ipv4.conf.all.forwarding = 1

# 首先，我们测试容器能否ping通外网
$ docker exec -ti nostalgic_agnesi ping www.baidu.com
PING www.baidu.com (61.135.169.125): 56 data bytes
64 bytes from 61.135.169.125: seq=0 ttl=51 time=62.042 ms

# 我们将ip_forward设置为0之后试试
$  sysctl net.ipv4.conf.all.forwarding=0
$ docker exec -ti nostalgic_agnesi ping www.baidu.com
^C
```
> **Note:** 注意，上面的设置不会影响到使用host网络的容器

大多数使用docker的人都希望`ip_forward`卡其，至少让容器和外网通信成为可能。但是如果你在一个多网桥中，也可能需要在容器中设置`ip_forward`哟。
除非你设置了`--iptables=false`，否则，docker会在启动的时候修改你的iptables规则，会添加`DOCKER`规则链。
docker是不会删除或修改任何之前的防火墙规则，这就允许了用户可以提前设定好对容器的一些访问规则。
默认情况下，docker是允许所有的外部IP，如果只想允许一个特定的ip或网段访问容器的话，可以在`DOCKER`规则链上添加新的规则。比如，限制是有源ip 8.8.8.8可以访问容器：
```bash
$ iptables -I DOCKER -i ext_if ! -s 8.8.8.8 -j DROP
```

# 容器间通信
容器间是否可以通信，由系统层的两个因素来控制：
* 网络的拓扑接口是否已经连接到容器的网络接口。默认情况下，docker会将所有的容器连接到docker0网桥，并为两个容器见包的传输提供路径。
* iptables是否允许特殊连接？除非你设置了`--iptables=false`，否则，docker会在启动的时候修改你的iptables规则，会添加`DOCKER`规则链。另外，如果你保留默认设置`--icc=true`,docker服务器或向FORWARD链添加一个带有全局ACCEPT策略的默认规则，如果不保留默认设置即`--icc=false`,系统会把策略设置成DROP

至于,是保留`--icc=true`,还是改变设置`--icc=false`，这是一个战略问题。如果设置了`--icc=false`，可以避免被攻击的容器来访问或扫描其他的主机和容器。
如果你选择了最安全的模式`--icc=false`，那么当你想让它们彼此提供服务的时候应该怎么让他们互相通信呢？
答案就是：前面提到的`--link=CONTAINER_NAME:ALIAS`选项。如果docker是在`--icc=false`和`--iptables=true`下运行，当它看到`docker run `调用`--link`选项，docker将会插入一部分iptables ACCEPT规则，使得心容器可以连接到其他容器暴露出的端口(即是Dockerfile中EXPOSE所指定的端口)

> **Note:** `--link`选项指定的CONTAINER_NAME，要么是运行`docker run`的时候自动生成的，要么是你在运行`docker run`时候指定`--name`的时候指定的。绝不能是主机名，否则在`--link`的上下文中是无法识别的。

我们可以在docker主机上运行`iptables`命令来看看，FORWARD规则链是否有默认的`ACCEPT`或`DROP`策略。
```bash
# 当设置--icc=false时
$ sudo iptables -L -n

...
Chain FORWARD (policy ACCEPT)
target     prot opt source               destination
DOCKER     all  --  0.0.0.0/0            0.0.0.0/0
DROP       all  --  0.0.0.0/0            0.0.0.0/0
...

# 当在--icc=false模式下，使用--link时
$ sudo iptables -L -n

...
Chain FORWARD (policy ACCEPT)
target     prot opt source               destination
DOCKER     all  --  0.0.0.0/0            0.0.0.0/0
DROP       all  --  0.0.0.0/0            0.0.0.0/0

Chain DOCKER (1 references)
target     prot opt source               destination
ACCEPT     tcp  --  172.17.0.2           172.17.0.3           tcp spt:80
ACCEPT     tcp  --  172.17.0.3           172.17.0.2           tcp dpt:80
```
