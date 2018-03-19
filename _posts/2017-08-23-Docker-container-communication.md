---
layout: post
title: Docker基础-理解容器之间的通信
categories: [docker]
description: 理解容器之间的通信
keywords: docker
---
本节主要是讲解docker容器是如何在默认的`bridge`网络上进行通信的。

<!--more-->

### 和外网通信

容器能否和外网进行通信，取决两个因素。

* 主机上的`ip_forward`是否打开，默认为true
* `iptables`规则是否允许被修改，默认为true

如果在启动 Docker 服务的时候设定 --ip-forward=true, Docker 就会自动设定系统的 ip_forward 参数为 1。如果你的系统上打开了`ip_froward`，而你在docker中设定`--ip-forward=flase`，这不会起作用。

关于`ip_froward`的实验

```bash
docker run -d --name  db  training/postgres

# 这时候首先去将ip_froward设置为false，发现无法ping通外网
 docker exec -ti db ping www.baidu.com

# 这时候再将ip_forward设置为true，发现可以ping通外网
docker exec -ti db ping www.baidu.com
PING www.a.shifen.com (103.235.46.39) 56(84) bytes of data.
64 bytes from 103.235.46.39: icmp_seq=1 ttl=54 time=56.3 ms
64 bytes from 103.235.46.39: icmp_seq=2 ttl=54 time=56.3 ms

```

默认情况下，docker会在启动容器的时候，在iptables中添加规则，如果你设定`--iptables=false`，docker将不会修改你的防火墙规则。

关于`--iptables`的实验

```bash
# 首先，确认下有没有关于docker0的防火墙规则，有就删除掉
iptables -S |grep docker0

# 然后测试ping外网，发现无法ping通
docker exec -ti db ping www.baidu.com

# 这时候将iptables改为true，查看防火墙规则，又添加了关于docker0的规则
iptables -S |grep docker
-A FORWARD -o docker0 -j DOCKER
-A FORWARD -o docker0 -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT
-A FORWARD -i docker0 ! -o docker0 -j ACCEPT
-A FORWARD -i docker0 -o docker0 -j ACCEPT

# 测试发现能ping通外网

docker exec -ti db ping www.baidu.com
PING www.a.shifen.com (103.235.46.39) 56(84) bytes of data.
64 bytes from 103.235.46.39: icmp_seq=1 ttl=54 time=56.3 ms
64 bytes from 103.235.46.39: icmp_seq=2 ttl=54 time=56.3 ms
```

默认情况下，容器可以和任何外部地址进行通信。除非你设定仅仅只有那些地址可以访问内部的容器。如下，设定只有8.8.8.8能访问内部容器

```bash
iptables -I DOCKER -i ext_if ! -s 8.8.8.8 -j DROP
```

### 容器之间通信

容器之间是否能进行通信，取决于系统层面的两个因素。

* 容器网络是否连接到`docker0`网络，默认是连接到docker0
* 你的`iptables`是否允许被修改，默认为true,你的`--icc`状态，默认为true

**注意：如果你使用默认的`iptables=true`，则当`--icc=true`时会添加 ACCEPT 规则，当`--icc=false`时会添加 DROP 规则; 当然如果你的`iptables=false`则无所谓了，反正不会修改你的防火墙规则**

关于`--icc`的实验

```bash
# 设置--icc=false发现容器之间无法ping通,如果你发现你依然能ping通，那可能是前面的防火墙配置，执行iptables -F 清空下再试
docker exec -ti db ping 172.17.0.3


# 设置--icc=true发现容器之间可以ping通
docker exec -ti db ping 172.17.0.3
PING 172.17.0.3 (172.17.0.3) 56(84) bytes of data.
64 bytes from 172.17.0.3: icmp_seq=1 ttl=64 time=0.128 ms
64 bytes from 172.17.0.3: icmp_seq=2 ttl=64 time=0.110 ms
64 bytes from 172.17.0.3: icmp_seq=3 ttl=64 time=0.096 ms

```

> **注意：所以，我们启动docker之前，就要确定好，到底是否需要允许docker修改iptables，是否允许容器之间通信，是否允许容器访问外网等**


### 主机之间的容器通信

官网上说的是将默认FORWARD DROP策略改为 ACCEPT，其实没那个必要，使用overlay网络就好了,比如:flannel,weave,calico等等...
