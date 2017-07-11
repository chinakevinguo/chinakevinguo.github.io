---
layout: post
title: Docker基础-docker muti-host之macvlan
categories: [docker]
description: docker muti-host之macvlan
keywords: docker
---
在这篇文章中，我将向您展示如何使用`macvlan`接口与Docker进行网络连接。使用`macvlan`接口为Docker容器提供一个有趣的网络配置，可能(取决于您的环境)地址与标准Linux网桥配置。`macvlan`接口，如果你不熟悉它们，它们是一个最近才添加到Linux内核的东西，使用户能够添加多个基于MAC地址的逻辑接口到单个物理接口。这些逻辑接口必须位于与关联的物理接口相同的广播域中，换句话说，Docker容器将在与主机相同的网络上(没有iptables规则，没有Linux网桥)，只是直接连接到主机的网络。

<!--more-->

Docker官网的解释是，`macvlan`驱动使Docker用户的使用实例和审查实施硬件化和就绪化，Libnetwork使用户能够完全控制IPv4和IPv6的寻址。Vlan驱动程序就是在此基础上构建，使运营商能够完全控制第二层的vlan tag。

`Macvlan`是一种新的尝试和真正的网络虚拟化技术。因为不是使用传统的Linux桥来进行隔离，所以`Macvlan`在linux上实现起来特别轻量级，它们只是关联到Linux物理接口，以实现网络之间的分离和到物理网络的连接。


`Macvlan`的各种模式，提供了很多的独有的功能，以及其提供了大量改进和创新的空间。这样有两个好处：
* 绕开网桥，提高了网络性能
* 可移动的部件少，更简单
移除了Docker主机和容器之间的接口，提升了外部访问容器的性能，因为这时候是没有了端口隐射的。

## 前提条件
* 配置了Docker engine 1.12.0+的单台主机
* 内核必须是V3.9-3.19和4.0+
* 使用子接口(如eth0.10)的示例都可以替换成eth0或者其他接口，子接口eth0.x是动态创建的，可以和`-o parent`指定的接口一起创建，并且驱动程序会创建虚拟接口，其将使本机能够保持连接

通过为物理网卡创建Macvlan子接口，允许一块物理网卡拥有多个独立的MAC地址和IP地址。虚拟出来的子接口将直接暴露在底层物理网络中。从外界看来，就像是把网线分成多股，分别接到了不同的主机上一样

## Macvlan 桥接模式例子
`Macvlan`桥接模式拥有每个容器唯一的MAC地址，用于追踪Docker主机的MAC到端口映射
* Macvlan 网络驱动连接到一个Docker主机的父接口。实例中是一个物理接口如eth0，一个用802.1q VLAN标记的子接口eth0.10(.10表示VLAN10)，或者捆绑两个以太网接口到一个逻辑接口的绑定主机适配器
* 一个外部的网关路由
* 每个Macvlan bridge 模式的Docker 网络彼此隔离，并且每次只能有一个网络连接到父接口，每个主机的适配器理论上可以连接4094个子接口
* 同一子网的任何容器可以与同一网络的任何其他容器进行通信
* `docker network`命令同样适用于vlan驱动
* 在`macvlan`模式下，如果两个网络/子网之间没有外部路由，那么两个网络上的容器是无法通信的，这一规则同样适用于同一docker网络的多个子网

在下面的例子中，都是在virtualbox中操作的
Route host： enp0s8:192.168.1.100(Hostonly模式) enp0s3:10.0.2.15(NAT模式，连接外网)
Docker host：enp0s8:192.168.1.1(Hostonly模式) gateway:192.168.1.100
![macvlan-bridge](/images/posts/macvlan-docker.png)
**Note:** Macvlan bridge模式下的子网必须和其所关联的网卡的网段相同，在这个例子中，使用`-o parent=enp0s8`来指定
1.配置router host的ip，以及路由转发等功能，使其成为一台路由
```bash
# 配置好IP
$ ip addr show
2: enp0s3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP qlen 1000
    link/ether 08:00:27:5a:e9:e7 brd ff:ff:ff:ff:ff:ff
    inet 10.0.2.15/24 brd 10.0.2.255 scope global dynamic enp0s3

3: enp0s8: <BROADCAST,MULTICAST,PROMISC,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP qlen 1000
    link/ether 08:00:27:ac:4f:00 brd ff:ff:ff:ff:ff:ff
    inet 192.168.1.100/24 brd 192.168.1.255 scope global enp0s8

# 开启路由转发,添加net.ipv4.ip_forward = 1
$ vi /etc/sysctl.conf
net.ipv4.ip_forward = 1

# 让路由转发即可生效
$ sysctl -p

# 配置SNAT，使所有来自192.168.1.0网段的包都交给10.0.2.15
$ iptables -t nat -A POSTROUTING -o enp0s3 -s 192.168.1.0/24 -j SNAT --to-source 10.0.2.15
```

2.最关键的一步，在两台主机上开启混杂模式（也许实体机不需要，没有测试过），记住virtualbox上的网卡也要开启混杂模式中的全部允许。
```bash
$ ip link set enp0s8 promisc on
```


3.在docker host上创建`macvlan`网络，然后运行一对容器
```bash
# 创建一个macvlan网络
$ docker network create -d macvlan \
      --subnet=192.168.1.0/24 \
      --gateway=192.168.1.100 \
      -o parent=enp0s8 macvlan_pub

# 在macvlan网络上运行一个容器并且使用`--ip`指定它的ip地址
$ docker run --network macvlan_pub --ip 192.168.1.201 -tid alpine /bin/sh
$ docker run --network macvlan_pub --ip 192.168.1.202 -tid alpine /bin/sh
```

登录一个容器查看IP
```bash
[root@node01 ~]# docker exec -ti desperate_raman ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
6: eth0@if3: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 1500 qdisc noqueue state UNKNOWN
    link/ether 02:42:c0:a8:01:c9 brd ff:ff:ff:ff:ff:ff
    inet 192.168.1.201/24 scope global eth0
       valid_lft forever preferred_lft forever
    inet6 fe80::42:c0ff:fea8:1c9/64 scope link
       valid_lft forever preferred_lft forever
```

查看容器路由
```bash
[root@node01 ~]# docker exec -ti desperate_raman route -n
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
0.0.0.0         192.168.1.100   0.0.0.0         UG    0      0        0 eth0
192.168.1.0     0.0.0.0         255.255.255.0   U     0      0        0 eth0
```

ping另一个容器和网关
```bash
[root@node01 ~]# docker exec -ti desperate_raman ping 192.168.1.202
    PING 192.168.1.202 (192.168.1.202): 56 data bytes
    64 bytes from 192.168.1.202: seq=0 ttl=64 time=0.140 ms
    64 bytes from 192.168.1.202: seq=1 ttl=64 time=0.127 ms

[root@node01 ~]# docker exec -ti desperate_raman ping 192.168.1.100
    PING 192.168.1.100 (192.168.1.100): 56 data bytes
    64 bytes from 192.168.1.100: seq=0 ttl=64 time=0.140 ms
    64 bytes from 192.168.1.100: seq=1 ttl=64 time=0.127 ms
```

**NOTE** 在macvlan网络中，容器是无法ping通docker主机上的IP的。

## Macvlan 802.1q trunk bridge 模式例子
VLAN(虚拟局域网)一直以来都是虚拟化数据中心网络的主要手段，目前任然在存在与大多数网络中。VLAN 是通过VLAN ID来实现广播隔离，网络运营商通常使用它将如web，db或其他需要隔离的应用进行隔离。
单台主机运行多个虚拟网络的情况是比较常见的，Linux网络支持VLAN标记(也称为802.1q)，用于维护网络之间的数据隔离。通过创建Linux子接口(每个自接口有自己唯一的VLAN ID)，使连接到docker主机以太网链路可以支持802.1q vlan id。
将802.1q中继到Linux宿主机是一件非常痛苦的事，它需要更改配置文件来保持持久化。如果涉及到网桥，还需要将物理网卡移动到网桥中，然后让网桥获取IP地址，在这一些列复杂的操作中，可能会导致服务器网络断开。

和所有其他的docker网络驱动程序一样，macvlan的主要目的是为了减少网络资源管理的操作性，所以，当网络收到一个不存在的子接口时，驱动会在创建网络的时候创建vlan 标记接口。

在宿主机重启的时候，驱动程序将在Docker守护程序重启的时候重新创建所有的网络连接，而不需要常常修改复杂的网络配置文件，驱动程序会跟踪，是否创建了最初网络创建时创建的带有VLAN 标记的子接口，并且在重启或者被删除后，会在原有的位置重新创建子接口（而且只会创建子接口）
如果用户不希望docker使用`-o parent`修改子接口，用户只需要传递哪些已经存在了的被作为父接口的接口即可，父接口不会被删除，只会删除没有master的子接口。
下面的例子就可以说明：
```bash
# 在docker宿主机上添加两个子接口
$ ip link add link enp0s8 name enp0s8.200 type vlan id 200
$ ip link add link enp0s8 name enp0s8.201 type vlan id 201
$ ip link set enp0s8.200 up
$ ip link set enp0s8.201 up

# 创建基于这两个子接口的macvlan
$ docker network create -d macvlan --subnet 192.168.200.0/24 --gateway=192.168.200.100 -o parent=enp0s8.200 macvlan200
$ docker network create -d macvlan --subnet 192.168.201.0/24 --gateway=192.168.201.100 -o parent=enp0s8.201 macvlan201

#重启宿主机，查看ip，发现没有刚刚创建的子接口
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
2: enp0s3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP qlen 1000
    link/ether 08:00:27:5a:e9:e7 brd ff:ff:ff:ff:ff:ff
3: enp0s8: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP qlen 1000
    link/ether 08:00:27:5f:be:76 brd ff:ff:ff:ff:ff:ff
    inet 192.168.1.1/24 brd 192.168.1.255 scope global enp0s8
       valid_lft forever preferred_lft forever
    inet6 fe80::a00:27ff:fe5f:be76/64 scope link
       valid_lft forever preferred_lft forever

# 启动docker之后再查看
$ systemctl start docker
$ ip addr show
valid_lft forever preferred_lft forever
5: enp0s8.200@enp0s8: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP
link/ether 08:00:27:5f:be:76 brd ff:ff:ff:ff:ff:ff
inet6 fe80::a00:27ff:fe5f:be76/64 scope link
valid_lft forever preferred_lft forever
6: enp0s8.201@enp0s8: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP
link/ether 08:00:27:5f:be:76 brd ff:ff:ff:ff:ff:ff
inet6 fe80::a00:27ff:fe5f:be76/64 scope link
valid_lft forever preferred_lft forever
```

下面用个完整的例子来说明
Docker host:
enp0s8:192.168.1.100(Hostonly模式)
enp0s3:10.0.2.15(NAT模式，连接外网)

Router host:
enp0s8:192.168.1.100
enp0s8.200:192.168.200.100
enp0s8.201:192.168.201.100
![multi-tenant-8021q-vlans](/images/posts/multi_tenant_8021q_vlans.png)

1.因为是使用的turnk模式，所以，我们首先在route host上新建几个vlan，并配上IP
```bash
# 安装工具
$ yum install vconfig
# 新建vlan
$ vconfig add enp0s8 200
$ vconfig add enp0s8 201
# 配置IP
$ ifconfig enp0s8.200 192.168.200.100 netmask 255.255.255.0 up
$ ifconfig enp0s8.201 192.168.201.100 netmask 255.255.255.0 up
# 开启混杂模式
$ ip link set enp0s8.200 promisc on
$ ip link set enp0s8.201 promisc on
# 配置SNAT
$ iptables -t nat -A POSTROUTING -o enp0s3 -s 192.168.200.0/24 -j SNAT --to-source 10.0.2.15
$ iptables -t nat -A POSTROUTING -o enp0s3 -s 192.168.201.0/24 -j SNAT --to-source 10.0.2.15
```
**NOTE** 上面配置的信息，如果想要保存的话，需要修改配置文件，这里是测试环境，所以，笔者我并没有保存
2.在Docker host上配置对应的子接口，并创建macvlan
```bash
# 在docker宿主机上添加两个子接口
$ ip link add link enp0s8 name enp0s8.200 type vlan id 200
$ ip link add link enp0s8 name enp0s8.201 type vlan id 201
$ ip link set enp0s8.200 up
$ ip link set enp0s8.201 up

# 创建基于这两个子接口的macvlan
$ docker network create -d macvlan --subnet 192.168.200.0/24 --gateway=192.168.200.100 -o parent=enp0s8.200 macvlan200
$ docker network create -d macvlan --subnet 192.168.201.0/24 --gateway=192.168.201.100 -o parent=enp0s8.201 macvlan201
```
3.运行容器
```bash
# 运行容器
$ docker run --net=macvlan200 -idt --ip=192.168.200.2 alpine /bin/sh
$ docker run --net=macvlan201 -idt --ip=192.168.201.2 alpine /bin/sh

$ docker ps
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS               NAMES
70448bfe282b        alpine              "/bin/sh"           19 hours ago        Up 15 hours                             gloomy_lalande
2b6b92345148        alpine              "/bin/sh"           19 hours ago        Up 15 hours                             gloomy_stonebraker
```
4.测试
```bash
# 查看容器ip
$ docker exec -ti gloomy_lalande ifconfig
eth0      Link encap:Ethernet  HWaddr 02:42:C0:A8:C9:02  
          inet addr:192.168.201.2  Bcast:0.0.0.0  Mask:255.255.255.0

$ docker exec -ti gloomy_stonebraker ifconfig
eth0      Link encap:Ethernet  HWaddr 02:42:C0:A8:C8:02  
          inet addr:192.168.200.2  Bcast:0.0.0.0  Mask:255.255.255.0

# 从201网段测试ping200网段
$ docker exec -ti gloomy_lalande ping 192.168.200.2
PING 192.168.200.2 (192.168.200.2): 56 data bytes
64 bytes from 192.168.200.2: seq=0 ttl=63 time=0.946 ms
64 bytes from 192.168.200.2: seq=1 ttl=63 time=1.490 ms

# traceroute一下，发现是先将包丢给了自己网段的网关后，再转发给200网段的，所以说，如果没有路由的话，就无法ping通了
$ docker exec -ti gloomy_lalande traceroute 192.168.200.2
traceroute to 192.168.200.2 (192.168.200.2), 30 hops max, 46 byte packets
 1  192.168.201.100 (192.168.201.100)  0.734 ms  0.589 ms  1.002 ms
 2  192.168.200.2 (192.168.200.2)  0.939 ms  1.076 ms  1.030 ms
```

通过上面的例子，我们发现，通过macvlan基本实现了vlan隔离，如果route host上没有200和201的话，两个vlan之间是完全无法通信的

`macvlan`先说到这里，更多了解，后续如果有新的内容，再补充吧。
