---
layout: post
title: Docker基础-docker自定义bridge网络
categories: [docker]
description: 自定义配置docker的bridge网络
keywords: docker
---
本章节主要讲了如何自定义docker默认的bridge网络，这个`bridge`网络是在docker安装的时候，系统默认创建的。
> **Note:** docker允许创建除了默认`bridge`网络之外的bridge网络

<!--more-->

默认情况下，docker服务器会创建和配置宿主机的docker0接口作为linux内核中的以太网网桥，可以在其他物理或虚拟接口之间来回床底数据包，使其作为单个以太网网络运行。
docker为 `docker0`配置IP地址，子网掩码和ip范围。主机可以为连接到`docker0`的容器发送和接受数据，并给他一个`MTU`(最大传输单元或最大包长-1500字节)，这些选项都是可以配置在docker的启动配置文件中：
* `--bip=CIDR` 为docker0配置IP地址
* `--fixed-cidr=CIDR` 限制docker0的子网范围,如：172.16.1.0/24，这个范围必须是包含了docker0 ip地址的范围
* `--mtu=BYTES` 设置docker0的mtu值

一旦你运行了一个或多个容器后，你可以使用`brctl`命令发现这些容器的接口都连接到了docker0网桥
```bash
$ brctl show
bridge name	bridge id		STP enabled	interfaces
bridge0		8000.86de2e92d296	no		veth79dcc4b
							vethc054462
```
如果你的机器上没有brctl这个命令， 你可以使用`yum install bridge-utils -y`(on centos)来进行安装。

最后，当你每次创建新容器的时候，都会使用`docker0`进行设置。docker会从docker0网桥的ip范围内，选择一个空闲的ip，分配给新容器，并且，会将docker0网络的IP地址，作为容器的网关：
```bash
# The network, as seen from a container

$ docker run -i -t --rm base /bin/bash

root@f38c87f2a42d:/# ip addr show eth0

24: eth0: <BROADCAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 32:6f:e0:35:57:91 brd ff:ff:ff:ff:ff:ff
    inet 172.17.0.3/16 scope global eth0
       valid_lft forever preferred_lft forever
    inet6 fe80::306f:e0ff:fe35:5791/64 scope link
       valid_lft forever preferred_lft forever

root@f38c87f2a42d:/# ip route

default via 172.17.42.1 dev eth0
172.17.0.0/16 dev eth0  proto kernel  scope link  src 172.17.0.3

root@f38c87f2a42d:/# exit
```

> **Note:** 记住，默认情况下，docker主机不会将容器包转发到外网，除非你设置`ip_forward`，具体可参考[communication to the outside world](https://docs.docker.com/engine/userguide/networking/default_network/container-communication/#communicating-to-the-outside-world)
