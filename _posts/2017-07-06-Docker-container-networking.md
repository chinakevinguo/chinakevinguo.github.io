---
layout: post
title: Docker基础-docker网络
categories: [docker]
description: docker默认的网络特性
keywords: docker
---
# Docker container networking

介绍docker网络常见的集中模式

<!--more-->

## 默认的网络模式
这部分主要介绍了Docker Engine本机提供的默认网络特性，描述了默认创建的网络类型，以及如何创建用户自定义网络，同时还描述了在单个主机或集群上创建网络需要哪些资源。
当你安装Docker的时候，Docker默认安装了三种网络，你可以使用`docker network ls`命令进行查看：
```bash
$ docker network ls

NETWORK ID          NAME                DRIVER
7fca4eb8c647        bridge              bridge
9f904ee27bf5        none                null
cf03ee007fb4        host                host
```

当运行容器的时候，可以使用`--network`来指定将容器运行在那个网络上

**bridge**网络模式，表示Docker默认安装的docker0网络，除非你通过`--network`进行了指定，否则Docker daemon会将所有容器都连接到docker0网络，你可以使用`ifconfig`命令对docker0网络进行查看：
```bash
$ ifconfig

docker0   Link encap:Ethernet  HWaddr 02:42:47:bc:3a:eb
          inet addr:172.17.0.1  Bcast:0.0.0.0  Mask:255.255.0.0
          inet6 addr: fe80::42:47ff:febc:3aeb/64 Scope:Link
          UP BROADCAST RUNNING MULTICAST  MTU:9001  Metric:1
          RX packets:17 errors:0 dropped:0 overruns:0 frame:0
          TX packets:8 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:0
          RX bytes:1100 (1.1 KB)  TX bytes:648 (648.0 B)
```

**none**网络模式，这种模式下，docker容器拥有自己的Network Namespace，但是，并不为Docker容器进行任何网络配置，也就是说，这个Docker容器没有网卡，IP，路由等信息，需要我们自己为Docker容器添加网卡，配置IP等。
```bash
$ docker attach nonenetcontainer

root@0cb243cd1293:/# cat /etc/hosts
127.0.0.1	localhost
::1	localhost ip6-localhost ip6-loopback
fe00::0	ip6-localnet
ff00::0	ip6-mcastprefix
ff02::1	ip6-allnodes
ff02::2	ip6-allrouters
root@0cb243cd1293:/# ifconfig
lo        Link encap:Local Loopback
          inet addr:127.0.0.1  Mask:255.0.0.0
          inet6 addr: ::1/128 Scope:Host
          UP LOOPBACK RUNNING  MTU:65536  Metric:1
          RX packets:0 errors:0 dropped:0 overruns:0 frame:0
          TX packets:0 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:0
          RX bytes:0 (0.0 B)  TX bytes:0 (0.0 B)
```

**host**网络模式，在这种模式下，docker没有自己的Network Namespace，而是和主机公用一个Network Namespace，容器不会虚拟出自己的网络卡，配置自己的IP，而是直接使用宿主机的IP和端口

## bridge网络的详细信息
除了默认的`bridge`网络外，我们一般不需要和其他的网络进行任何交互，这些默认的网络模式，你不能删除，但是你可以使用`docker network inspect`来进行查看。
```bash
[root@vultr ~]# docker network inspect bridge
[
    {
        "Name": "bridge",
        "Id": "83ffb543e3074f0071a84e809894d1d033211afa34100b088e4dd23c219f44e6",
        "Scope": "local",
        "Driver": "bridge",
        "EnableIPv6": false,
        "IPAM": {
            "Driver": "default",
            "Options": null,
            "Config": [
                {
                    "Subnet": "172.17.0.0/16",
                    "Gateway": "172.17.0.1"
                }
            ]
        },
        "Internal": false,
        "Containers": {
            "acc26c4abbd15c6bede466be22cfc3622c3e5e078d7de16a57f411ea1606f435": {
                "Name": "container1",
                "EndpointID": "d88640837d9f5a373e4fe44aa2c67ff503dea7db60085fa0d0a22598d226d065",
                "MacAddress": "02:42:ac:11:00:03",
                "IPv4Address": "172.17.0.3/16",
                "IPv6Address": ""
            },
            "c4fd7144e3ad4426fdc9536e910efd1ad5dfd4fbc76d2d0b4c619e50dc616a7d": {
                "Name": "vps-server",
                "EndpointID": "c425566b3cbfdea5caf8d1790d8f75535d33c2eae109e674663e6651d64e0f66",
                "MacAddress": "02:42:ac:11:00:02",
                "IPv4Address": "172.17.0.2/16",
                "IPv6Address": ""
            },

        "Options": {
            "com.docker.network.bridge.default_bridge": "true",
            "com.docker.network.bridge.enable_icc": "true",
            "com.docker.network.bridge.enable_ip_masquerade": "true",
            "com.docker.network.bridge.host_binding_ipv4": "0.0.0.0",
            "com.docker.network.bridge.name": "docker0",
            "com.docker.network.driver.mtu": "1500"
        },
        "Labels": {}
    }
]

```
上面的`docker network inspect`命令列出了给定网络上的所有连接的容器和网络资源。
容器之间默认是使用IP地址进行通信，bridge网络模式不支持服务自动发现，如果你想要在bridge网络模式下容器之间通过容器名进行通信，你需要使用`docker run --link`来连接容器。

## 用户自定义网络
为了更好的隔离和操作容器，用户可以创建自己的自定义网络，为此，Docker提供了许多默认的网络驱动，包括**bridge network**,**overlay network**,**MACVLAN network**,你甚至可以自己来编写网络插件来操作自己的网络。

用户可以创建多个网络，也可以将容器添加到多个网络，连接两个网络的容器可以与任意一个网络中的成员容器进行通信，但是，两个网络中的容器是不能进行通信的，所以说，容器只能在网络中通信，而不能在网络间通信（也就是说，容器只能在同一网络中通信）。
### A bridge network
创建自定义网络最简单的是创建bridge网络
```bash
[root@vultr ~]$ docker network create --driver bridge isolated_nw
8a1c90ca9772d4b9be680f503a1aa9c8cabb660a28f947bb3e37519bc6cf5f45

[root@vultr ~]$ docker network inspect isolated_nw
[
    {
        "Name": "isolated_nw",
        "Id": "8a1c90ca9772d4b9be680f503a1aa9c8cabb660a28f947bb3e37519bc6cf5f45",
        "Scope": "local",
        "Driver": "bridge",
        "EnableIPv6": false,
        "IPAM": {
            "Driver": "default",
            "Options": {},
            "Config": [
                {
                    "Subnet": "172.18.0.0/16",
                    "Gateway": "172.18.0.1/16"
                }
            ]
        },
        "Internal": false,
        "Containers": {},
        "Options": {},
        "Labels": {}
    }
]
```
当你创建了网络后，你可以使用`docker run  --network=<NETWORK>`选项来启动容器
```bash
[root@vultr ~]$ docker run --network=isolated_nw -tid --name=container1 busybox
4d84de8318d4131fcb6d86a29a916d5354111667806fa15f6c5e3f9589766416

[root@vultr ~]$ docker network inspect isolated_nw
[
    {
        "Name": "isolated_nw",
        "Id": "8a1c90ca9772d4b9be680f503a1aa9c8cabb660a28f947bb3e37519bc6cf5f45",
        "Scope": "local",
        "Driver": "bridge",
        "EnableIPv6": false,
        "IPAM": {
            "Driver": "default",
            "Options": {},
            "Config": [
                {
                    "Subnet": "172.18.0.0/16",
                    "Gateway": "172.18.0.1/16"
                }
            ]
        },
        "Internal": false,
        "Containers": {
            "4d84de8318d4131fcb6d86a29a916d5354111667806fa15f6c5e3f9589766416": {
                "Name": "container1",
                "EndpointID": "77e6674aaec2441d078b010469f96510d6be21f2f552c0cc14e2650417e342c7",
                "MacAddress": "02:42:ac:12:00:02",
                "IPv4Address": "172.18.0.2/16",
                "IPv6Address": ""
            }
        },
        "Options": {},
        "Labels": {}
    }
]

```
你在此网络启动的容器，必须是和该网络在同一台Docker主机上，虽然网络本身将容器与外部网络隔离开，但是在此网络中的容器，还是能立即与该网络中的其他容器进行通信，如下图：
![bridge_network](/images/posts/bridge_network.png)
在用户自定义的网络中，不支持link。如果你想要使bridge的一部分网络可以访问外网，你可以在此网络中的容器上公开和发布容器端口。
![network_access](/images/posts/network_access.png)

**bridge**网络在单个主机上运行比较有用，但是，在大规模网络中，一般使用**overlay**网络模式

### The docker_gwbridge network
`docker_gwbridge`是一个本地的bridge network，一般在如下两种情况下，Docker会自动创建：
> * 初始化或者加入swarm,Docker会创建`docker_gwbridge`网络，并使用它在不同主机节点间进行通信
> * 当容器的网络都不能提供外部链接时，Docker会将容器链接到`docker_gwbridge`网络以及容器的其他网络，以便容器能连接到外部网络或其他swarm节点

如果你需要自定义配置，你可以提前创建`docker_gwbridge`网络，否则Docker会按需创建，下面的例子介绍了，如何创建一个带有自定义选项的`docker_gwbridge`网络：
```bash
$ docker network create --subnet 172.30.0.0/16 \
                        --opt com.docker.network.bridge.name=docker_gwbridge \
                        --opt com.docker.network.bridge.enable_icc=false \
                        docker_gwbridge
```

### An overlay network with Docker Engine swarm mode without key-value store
你可以在没有外部key-value存储的情况下，在swarm的管理节点上创建`overlay`网络，swarm集群使`overlay`网络仅对集群中需要服务的节点可用，当你创建了一个在`overlay`网络覆盖下的服务时，管理节点会自动扩展`overlay`到运行该服务的节点上。

想了解更多有关`Docker swarm`的信息，可参考[Swarm mode overview](https://docs.docker.com/engine/swarm/)

下面的例子介绍了如何创建`overlay`网络，并将其用于swarm集群中的服务节点：
```bash
# Create an overlay network `my-multi-host-network`.
$ docker network create \
  --driver overlay \
  --subnet 10.0.9.0/24 \
  my-multi-host-network

400g6bwzd68jizzdx5pgyoe95

# Create an nginx service and extend the my-multi-host-network to nodes where
# the service's tasks run.
$ docker service create --replicas 2 --network my-multi-host-network --name my-web nginx

716thylsndqma81j6kkkb5aus
```
`overlay`网络不能应用于不属于swarm集群服务的容器，更多信息可参考[Docker swarm mode overlay network security model](https://docs.docker.com/engine/userguide/networking/overlay-security-model/)

### An overlay network with an external key-value store
如果你没有使用swarm，那么`overlay`网络可能会需要一个`key-value`的存储服务。支持以`key-value`方式存储的服务包括`Cousul`,`Etcd`,`ZooKeeper`...所以在此之前，你先要创建和配置你选择的`key-value`store，保证docker host和key-value store能够互相通信。

> * **注意：** 集群模式下的Docker Engine和使用外部存储的网络不兼容
![](/images/posts/key_value.png)
在网络中的每个主机都必须运行一个Docker Engine实例，提供主机最简单的方法是使用Docker Machine
![](/images/posts/engine_on_net.png)

你还应该在每个主机上都打开如下的端口：

| Protocol | Port | Description |
| :--------: | ---- | :----------- |
| udp | 4789 | Data plane[VXLAN] |
| tcp/upd | 7946 | Control plane |

你的key-value store服务如果还需要打开其他端口，可以参考服务供应商提供给你的文档进行操作
如果你还准备使用加密方式[--opt encrypted]，你还需要打开50[ESP]端口
一旦你配置了多态机器，你可以使用Docker swarm将他们快速的组成一个包含服务自动发现的集群。
创建`overlay`网络，你可以在每个你需要使用`overlay`网络`Docker Engine`上配置如下选项：

| Option | Description |
| :----- | :---------: |
| --cluster-store=PROVIDER://URL | Describes the location of the kv service |
| --cluster-advertise=HOST_IP/HOST_IFACE:PORT | The IP address of the HOST used for clustering |
| --cluster-store-opt=KEY-VALUE OPTIONS | Options such as TLS certificate or tuning discovery Timers |

在docker swarm中的某一台机器上创建`overlay`网络：
```bash
$ docker network create --driver overlay my-multi-host-network
```

创建之后，`overlay`网络会覆盖多个主机，网络范围下的容器会与其他网络的主机完全隔离
![overlay_network](/images/posts/overlay_network.png)
然后，在每个host上启动容器的时候记住都要制定network name
```bash
$ docker run -tid --network=my-multi-host-network busybox
```

一旦连接，`overlay`网络范围内的所有容器都可以互相访问，而不管该容器是在哪个主机上
![overlay-network-final](/images/posts/overlay-network-final.png)

如果你想自己试试，可以参考[Getting started for overlay](https://docs.docker.com/engine/userguide/networking/get-started-overlay/)

### 自定义网络插件
If you like ,you can write your own network driver plugin.网络驱动插件都是以Docker的基础插件为基础,if you like and you can do it ,you can refer [Extending Docker](https://docs.docker.com/engine/extend/legacy_plugins/) and [writing a network driver plugin](https://docs.docker.com/engine/extend/plugins_network/)

### Docker 内嵌的DNS server
Docker daemon 运行一个内嵌的DNS服务来为链接到自定义网络的容器提供服务自动发现，来自容器的名称解析首先由内嵌的DNS服务进行解析，如果内置的DNS无法解析，那么就转发给为容器配置的任意外部DNS服务器，在`resolv.conf`文件中，可看到`127.0.0.11`DNS，更多关于内嵌DNS的信息可参考[embedded DNS server in user-defined](https://docs.docker.com/engine/userguide/networking/configure-dns/)

## Links
在这之前，我们使用`docker link`来让容器之间进行通信。通过引入docker network，容器可以通过主机名进行通信，但是，你仍然可以使用`docker link`，但是只能在docker0网桥中使用。更多有关信息，可以参考[Legacy Links](https://docs.docker.com/engine/userguide/networking/default_network/dockerlinks/)和[linking containers in user-defined networks](https://docs.docker.com/engine/userguide/networking/work-with-networks/#linking-containers-in-user-defined-networks)
