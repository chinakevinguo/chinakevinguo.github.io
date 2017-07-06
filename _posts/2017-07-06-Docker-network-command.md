---
layout: post
title: Docker基础-docker network常用命令
categories: [docker]
description: docker network常用命令
keywords: docker
---
本文提供了几个Docker network和container进行交互的命令的示例。这些命令可通过CLI提供，这些命令是：
* `docker network create`
* `docker network connect`
* `docker network ls`
* `docker network rm`
* `docker network disconnect`
* `docker network inspect`

<!--more-->
虽然不是必须，但是在尝试这些示例之前，你最好先看看[Understanding Docker network](https://docs.docker.com/engine/userguide/networking/).这些例子都是依靠`bridge`网络，因此这些例子都可以马上执行。当然，如果你想要尝试`overlay`网络示例，你最好先看看[Getting started with multi-host networks](https://docs.docker.com/engine/userguide/networking/get-started-overlay/)

# Create networks
当你安装Docker的时候会自动安装`bridge`网络，这个`bridge`网络对应的是网卡上的`docker0`网络，除了这个`bridge`网络，你还可以创建自己的`bridge`网络或者`overlay`网络。

`bridge`网络运行在一个运行了`docker engine`的单独主机上，而`overlay`网络则可以覆盖到多个运行了`docker engine`的主机上。如果你运行`docker network create`创建了一个`bridge`网络，并且给了它一个名字，那么你可以通过`docker inspect`命令来进行查看：
```bash
$ docker network create simple-network
69568e6336d8c96bbf57869030919f7c69524f71183b44d80948bd3927c87f6a

$ docker network inspect simple-network
[
    {
        "Name": "simple-network",
        "Id": "69568e6336d8c96bbf57869030919f7c69524f71183b44d80948bd3927c87f6a",
        "Scope": "local",
        "Driver": "bridge",
        "IPAM": {
            "Driver": "default",
            "Config": [
                {
                    "Subnet": "172.22.0.0/16",
                    "Gateway": "172.22.0.1/16"
                }
            ]
        },
        "Containers": {},
        "Options": {}
    }
]
```

和`bridge`网络不一样，`overlay`网络的创建，需要提前做很多准备工作，准备工作如下：
* 有一个可以访问的`key-value`存储，如：Cousul，Etcd，Zookeeper等
* 有一个可以访问`key-value`存储的集群
* 在集群中的每个主机上配置正确运行的`docker daemon`

`dockerd`支持运行`overlay`网络的选项有：
* --cluster-store
* --cluster-store-opt
* --cluster-advertise

默认情况下，当你创建网络的时候，Docker engine会创建一个不重复的子网，你可以覆盖这个默认值，并通过`--subnet`选项来自定义子网，在`bridge`网络中，你只能指定单个子网，而在`overlay`网络中，你可以指定多个子网。

> * **Note:** 强烈建议使用`--subnet`来创建网络，如果没有指定`--subnet`，docker dameon会自动选择一个子网，但是这有可能会和其他没有被docker 管理的子网冲突，而当容器链接到这个网络的时候，子网重叠则会出现各种问题。
除了`--subnet`选项之外，你还可以指定`--gateway`,`--ip-range`,`--aux-address`等选项：
```bash
$ docker network create -d overlay \
  --subnet=192.168.0.0/16 \
  --subnet=192.170.0.0/16 \
  --gateway=192.168.0.100 \
  --gateway=192.170.0.100 \
  --ip-range=192.168.1.0/24 \
  --aux-address="my-router=192.168.1.5" --aux-address="my-switch=192.168.1.6" \
  --aux-address="my-printer=192.170.1.5" --aux-address="my-nas=192.170.1.6" \
  my-multihost-network
```
请确认你的网络没有冲突，如果冲突了，网络创建会出错，docker engine会返回错误值。

当创建自定义网络时，可以传递一些选项给驱动，`bridge`网络可以传递如下选项：

| Option | Equivalent | Description |
| :----- | :--------: | :---------- |
| com.docker.network.bridge.name | - | 创建`bridge`网络时的名称 |
| com.docker.network.bridge.enable_ip_masquerade | --ip-masq | 开启网络地址转换 |
| com.docker.network.bridge.enable_icc | --icc | 开启或禁用容器间的链接 |
| com.docker.network.bridge.host_binding_ipv4 | --ip | 隐射容器端口时的默认IP |
| com.docker.network.driver.mtu | --mtu | Set the containers network MTU |

`com.docker.network.driver.mtu`选项也支持`overlay`网络驱动
当使用`docker network create`创建网络时，下面的参数可以床底给任何网络：
| Argument | Equivalent | Description |
| :------ :| :--------: | :---------- |
| --internal | - | 限制访问外部网络  |
| --ipv6 | --ipv6 | 开启ipv6网络 |

下面的例子是演示，使用`-o`选项来指定在运行容器并指定隐射端口时的默认IP地址，然后使用`docker network inspect`来查看
```bash
$ docker network create -o "com.docker.network.bridge.host_binding_ipv4"="172.23.0.1" --subnet 172.23.0.0/16 my-network

b1a086897963e6a2e7fc6868962e55e746bee8ad0c97b54a5831054b5f62672a

$ docker network inspect my-network

[
    {
        "Name": "my-network",
        "Id": "b1a086897963e6a2e7fc6868962e55e746bee8ad0c97b54a5831054b5f62672a",
        "Scope": "local",
        "Driver": "bridge",
        "IPAM": {
            "Driver": "default",
            "Options": {},
            "Config": [
                {
                    "Subnet": "172.23.0.0/16",
                    "Gateway": "172.23.0.1/16"
                }
            ]
        },
        "Containers": {},
        "Options": {
            "com.docker.network.bridge.host_binding_ipv4": "172.23.0.1"
        }
    }
]

$ docker run -d -P --name redis --network my-network redis

bafb0c808c53104b2c90346f284bda33a69beadcab4fc83ab8f2c5a4410cd129

$ docker ps
#这里可以看到我在运行容器的时候使用-P选项隐射容器内部端口到宿主机的时候，默认的IP就是我刚刚指定的那个IP地址
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS                        NAMES
bafb0c808c53        redis               "/entrypoint.sh redis"   4 seconds ago       Up 3 seconds        172.23.0.1:32770->6379/tcp   redis

$ ifconfig
br-7a154192d2a8: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 172.23.0.1  netmask 255.255.0.0  broadcast 0.0.0.0
        inet6 fe80::42:b3ff:fe4a:d59f  prefixlen 64  scopeid 0x20<link>
        ether 02:42:b3:4a:d5:9f  txqueuelen 0  (Ethernet)
        RX packets 8  bytes 648 (648.0 B)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 16  bytes 1296 (1.2 KiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

```

# Connect containers
## Basic container networking example
你可以将容器连接到一个或多个网络，一个容器可以连接到多个使用不同网络驱动的网络，一旦连接之后，容器之间可以使用IP地址或容器名进行通信。
`overlay`网络或其他自定义的网络插件，哪怕容器在不同的主机上启动，只要支持能连接到多主机网络，就能够通过这种方式进行通信。通过下面的例子来看看：

1.运行两个容器，`container1`和`container2`
```bash
$ docker run -itd --name=container1 busybox

18c062ef45ac0c026ee48a83afa39d25635ee5f02b58de4abc8f467bcaa28731

$ docker run -itd --name=container2 busybox

498eaaaf328e1018042c04b2de04036fc04719a6e39a097a4f4866043a2c2152
```
2.创建一个名为isolated_nw的`bridge`网络
```bash
$ docker network create -d bridge --subnet 172.25.0.0/16 --gateway 172.25.0.1 isolated_nw

06a62f1c73c4e3107c0f555b7a5f163309827bfbbf999840166065a8f35455a8
```
3.连接`container2`到上面创建的这个网络，并使用`docker network inspect`进行查看
```bash
$ docker network connect isolated_nw container2

$ docker network inspect isolated_nw

[
    {
        "Name": "isolated_nw",
        "Id": "06a62f1c73c4e3107c0f555b7a5f163309827bfbbf999840166065a8f35455a8",
        "Scope": "local",
        "Driver": "bridge",
        "IPAM": {
            "Driver": "default",
            "Config": [
                {
                    "Subnet": "172.25.0.0/16",
                    "Gateway": "172.25.0.1/16"
                }
            ]
        },
        "Containers": {
            "90e1f3ec71caf82ae776a827e0712a68a110a3f175954e5bd4222fd142ac9428": {
                "Name": "container2",
                "EndpointID": "11cedac1810e864d6b1589d92da12af66203879ab89f4ccd8c8fdaa9b1c48b1d",
                "MacAddress": "02:42:ac:19:00:02",
                "IPv4Address": "172.25.0.2/16",
                "IPv6Address": ""
            }
        },
        "Options": {}
    }
]
```
我们发现`container2`已经被自动分配了一个IP地址，这是因为你在创建网络的时候指定了`--subnet`选项，因此在容器连接到网络的时候会自动从这个子网中选择IP分配。
4.启动第三个容器，但是使用`--ip`来连接到`isolated_nw`网络
```bash
$ docker run --network=isolated_nw --ip=172.25.3.3 -itd --name=container3 busybox

467a7863c3f0277ef8e661b38427737f28099b61fa55622d6c30fb288d88c551
```
只要你为容器指定的IP地址是网络子网的一部分，就可以使用`--ip`或`--ip6`来为该容器指定IP地址，当您使用用户自定义网络时用这种方法来指定IP地址，配置将会作为容器的一部分保留，并在容器重新加载的时候应用，如果您使用的不是用户自定义玩过的时候，Docker容器重启之后，容器内的子网可能会丢失。
5.查看`container3`的网络配置
```bash
$ docker inspect --format=''  container3

{"isolated_nw":
  {"IPAMConfig":
    {
      "IPv4Address":"172.25.3.3"},
      "NetworkID":"1196a4c5af43a21ae38ef34515b6af19236a3fc48122cf585e3f3054d509679b",
      "EndpointID":"dffc7ec2915af58cc827d995e6ebdc897342be0420123277103c40ae35579103",
      "Gateway":"172.25.0.1",
      "IPAddress":"172.25.3.3",
      "IPPrefixLen":16,
      "IPv6Gateway":"",
      "GlobalIPv6Address":"",
      "GlobalIPv6PrefixLen":0,
      "MacAddress":"02:42:ac:19:03:03"}
    }
  }
}
```
上面可以看到我们指定的IP地址，因为你在启动的时候将`container3`连接到了`isolated_nw`网络，并且指定了IP，所以它不再会连接到默认的`bridge`网络
6.查看container2的网络信息
```bash
$ docker inspect --format=''  container2 | python -m json.tool

{
    "bridge": {
        "NetworkID":"7ea29fc1412292a2d7bba362f9253545fecdfa8ce9a6e37dd10ba8bee7129812",
        "EndpointID": "0099f9efb5a3727f6a554f176b1e96fca34cae773da68b3b6a26d046c12cb365",
        "Gateway": "172.17.0.1",
        "GlobalIPv6Address": "",
        "GlobalIPv6PrefixLen": 0,
        "IPAMConfig": null,
        "IPAddress": "172.17.0.3",
        "IPPrefixLen": 16,
        "IPv6Gateway": "",
        "MacAddress": "02:42:ac:11:00:03"
    },
    "isolated_nw": {
        "NetworkID":"1196a4c5af43a21ae38ef34515b6af19236a3fc48122cf585e3f3054d509679b",
        "EndpointID": "11cedac1810e864d6b1589d92da12af66203879ab89f4ccd8c8fdaa9b1c48b1d",
        "Gateway": "172.25.0.1",
        "GlobalIPv6Address": "",
        "GlobalIPv6PrefixLen": 0,
        "IPAMConfig": null,
        "IPAddress": "172.25.0.2",
        "IPPrefixLen": 16,
        "IPv6Gateway": "",
        "MacAddress": "02:42:ac:19:00:02"
    }
}
```
我们可以看到`container2`同时属于两个网络，当你启动它的时候默认加入了`bridge`网络，然后在第三步的时候，你将它加入了`isolated_nw`网络，所以如下图所示：
![working](/images/working.png)
```bash
$ docker exec -ti container2 ifconfig
eth0      Link encap:Ethernet  HWaddr 02:42:AC:11:00:04  
          inet addr:172.17.0.4  Bcast:0.0.0.0  Mask:255.255.0.0
          inet6 addr: fe80::42:acff:fe11:4/64 Scope:Link
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:13 errors:0 dropped:0 overruns:0 frame:0
          TX packets:8 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:0
          RX bytes:982 (982.0 B)  TX bytes:648 (648.0 B)

eth1      Link encap:Ethernet  HWaddr 02:42:AC:19:00:02  
          inet addr:172.25.0.2  Bcast:0.0.0.0  Mask:255.255.0.0
          inet6 addr: fe80::42:acff:fe19:2/64 Scope:Link
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:32 errors:0 dropped:0 overruns:0 frame:0
          TX packets:8 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:0
          RX bytes:2592 (2.5 KiB)  TX bytes:648 (648.0 B)

```
7.Docker 内置的DNS服务会为连接到自定义网络的容器提供解析，这意味着任何链接到同样自定义网络的容器之间可以通过容器名进行通信。例子如下：
```bash
# 我们进入container2，container2即属于isolated_nw网络，也属于bridge网络
$ docker exec -ti container2 /bin/sh

#  ping我们刚刚启动的container3，可以ping通，因为container3是连接到自定义网络isolated_nw
/ # ping -w 4 container3
PING container3 (172.25.3.3): 56 data bytes
64 bytes from 172.25.3.3: seq=0 ttl=64 time=0.070 ms
64 bytes from 172.25.3.3: seq=1 ttl=64 time=0.080 ms
64 bytes from 172.25.3.3: seq=2 ttl=64 time=0.080 ms
64 bytes from 172.25.3.3: seq=3 ttl=64 time=0.097 ms

# 而我们ping刚刚启动的container1，是无法ping通的，因为container1是连接到默认的bridge网络的
/ # ping container1
ping: bad address 'container1'

# 我们再将container1和isolated_nw网络连接起来
$ docker network connect isolated_nw container1

#接着再ping container1，发现是可以ping通的
/ # ping container1
PING container1 (172.25.0.4): 56 data bytes
64 bytes from 172.25.0.4: seq=0 ttl=64 time=0.250 ms
64 bytes from 172.25.0.4: seq=1 ttl=64 time=0.149 ms
64 bytes from 172.25.0.4: seq=2 ttl=64 time=0.102 ms
```
上面的例子告诉我们，如果要使内置的DNS起作用，看到效果，我们应该使用自定义网络，而不是使用默认网络。

8.当前，`container2`同时属于`isolated_nw`和`bridge`网络，所以它可以和`container1`和`container3`进行通信，其中和`container1`通过IP进行通信，而`container3`可以通过IP和容器名进行通信，然而`container1`和`container3`没有任何相同的网络，所以它们之间是不能进行通信的。
```bash
$ docker attach container3

$ ping 172.17.0.2
PING 172.17.0.2 (172.17.0.2): 56 data bytes
^C

--- 172.17.0.2 ping statistics ---
10 packets transmitted, 0 packets received, 100% packet loss
```
## Linking container without using user-defined networks
在我们完成了前面的[Basic container networking examples](https://docs.docker.com/engine/userguide/networking/work-with-networks/#basic-container-networking-examples)例子后，`container2`和`container3`之间因为是通过自定义网络`isolated_nw`网络连接，所以可以通过容器名进行通信，然而那些通过默认的`bridge`网络连接的容器则不可以，那么应该怎么办呢，这里，我们使用`--link`来允许连接到`bridge`网络的容器通过容器名进行通信。

> **NOTE:** 官方强烈建议，不使用`--link`来允许容器间通过容器名通信，而使用自定义网络来允许容器见通过容器名进行通信

通过传统的`link`在默认`bridge`网络上通信，会添加如下功能：
> * 解析容器名到IP
> * 为将要连接到的容器定义一个别名，`--link CONTAINER-NAME:ALIAS`
> * 安全的容器连接[通过--icc=false进行隔离]
> * 环境变量注入

> **再次重申：** 所有上面的功能都能通过自定义网络实现，而且自定义网络还能动态附加到多个网络或从多个网络分离
> * 使用DNS自动名称解析
> * 使用`--link`为连接的容器提供别名
> * 为网络中的容器自动提供安全隔离
> * 环境变量注入

下面的例子简要的介绍了如何使用`--link`
1.运行一个`container1`容器，在启动`container2`的时候使用`--link`对`container2`和`container1`进行连接
```bash
$ docker run -tid --name container1 busybox
$ docker run -tid --name container2 --link container1:c1 busybox
```
2.我们在`container2`来ping下`container1`或`c1`试试
```bash
$ docker exec -ti container2 ping container1
PING container1 (172.17.0.3): 56 data bytes
64 bytes from 172.17.0.3: seq=0 ttl=64 time=0.294 ms
64 bytes from 172.17.0.3: seq=1 ttl=64 time=0.116 ms
64 bytes from 172.17.0.3: seq=2 ttl=64 time=0.142 ms
64 bytes from 172.17.0.3: seq=3 ttl=64 time=0.108 ms

$ docker exec -ti container2 ping c1
PING c1 (172.17.0.3): 56 data bytes
64 bytes from 172.17.0.3: seq=0 ttl=64 time=0.206 ms
64 bytes from 172.17.0.3: seq=1 ttl=64 time=0.125 ms
64 bytes from 172.17.0.3: seq=2 ttl=64 time=0.097 ms
64 bytes from 172.17.0.3: seq=3 ttl=64 time=0.122 ms
```
3.我们在`container1`下来ping`container2`试试
```bash
$ docker exec -ti container1 ping container2
ping: bad address 'container2'
```
我们发现只能在`container2`下能ping通`container1`，这说明，在`bridge`网络下，只有使用`--link`进行连接的容器之间能够通过主机名或别名进行互相通信，且只能是要主动连接的容器`container2`可以ping通被连接的容器`container1`

## 别名示例

当你使用link的时候，无论是使用传统的`link`，还是使用自定义网络的方式，任何别名都只会对你所指定的容器生效，而不会影响到其他的容器，此外，如果容器属于多个网络，你所给定的别名也只会你所限定的网络中生效。因此，容器可以连接到不同的网络使用不同的别名。请看下面的例子：
1.创建一个另外的网络`local_alias`
```bash
$ docker network create -d bridge --subnet 172.26.0.0/24 local_alias
```
2.将容器连接到新建的网络
```bash
$ docker network connect --link container1:foo local_alias container2
$ docker network connect --link container2:bar local_alias container1
```
3.我们进入container2，来进行测试
```bash
$ docker exec -ti container2 /bin/sh
/ # ping c1
PING c1 (172.17.0.3): 56 data bytes
64 bytes from 172.17.0.3: seq=0 ttl=64 time=0.263 ms

/ # ping foo
PING foo (172.26.0.3): 56 data bytes
64 bytes from 172.26.0.3: seq=0 ttl=64 time=0.134 ms
```
4.断开`container1`和`local_alias`网络的连接，再来测试
```bash
/ # ping c1
PING c1 (172.17.0.3): 56 data bytes
64 bytes from 172.17.0.3: seq=0 ttl=64 time=0.263 ms

/ # ping foo
ping: bad address 'foo'
```
你会发现，我依然能ping通c1,这是因为c1是我在连接`bridge`网络时指定的别名，而我无法ping通foo，这是因为foo是我在连接`local_alias`时指定的别名，现在我断开了和`local_alias`网络的连接， 当然就不能ping通了

## docker 网络限制
虽然我们建议使用docker network来控制你的容器，但是目前docker的网络在使用上还是有些限制的
## 环境变量注入
环境变量注入是静态的，而且一旦容器启动就无法更改了，传统的`--link`会将所有的环境变量共享给被连接的容器，而`docker network`则不会，当你使用`docker network`连接时，容器之间是不能动态的存在环境变量的。
## 网络别名
传统的links是提供的单向隔离以及单向别名，而网network-scoped会为所有的网络成员提供别名，我们来看看下面的例子：
1.先启动一个连接到`isolated_nw`网络的容器`container2`,并为它设定别名`app1`
```bash
$ docker run -tid --network isolated_nw --network-alias app1 --name container2 busybox
```
2.再启动一个连接到`local_alias`网络的容器`container3`，并为它设定别名`app2`
```bash
$ docker run -tid --network local_alias --network-alias app2 --name container3 busybox
```
3.将`container2`通过连接到`local_alias`网络，并为它设定别名`app3`
```bash
$ docker network connect --alias app3 local_alias container2
```
4.进入container2测试
```bash
#进入container2
/ # ping app1
PING app1 (172.19.0.2): 56 data bytes
64 bytes from 172.19.0.2: seq=0 ttl=64 time=0.059 ms

/ # ping app2
PING app2 (172.18.0.2): 56 data bytes
64 bytes from 172.18.0.2: seq=0 ttl=64 time=0.396 ms

/ # ping app3
PING app3 (172.18.0.3): 56 data bytes
64 bytes from 172.18.0.3: seq=0 ttl=64 time=0.069 ms
```
5.进入container3测试
```bash
/ # ping app1
ping: bad address 'app1'

/ # ping app2
PING app2 (172.18.0.2): 56 data bytes
64 bytes from 172.18.0.2: seq=0 ttl=64 time=0.117 ms

/ # ping app3
PING app3 (172.18.0.3): 56 data bytes
64 bytes from 172.18.0.3: seq=0 ttl=64 time=0.125 ms
```
我们发现在`container2`中能ping通所有，而在`container3`中无法ping通`app1`，这是因为`app1`属于`isolated_nw`网络，而`container3`只属于`local_alias`网络，所以无法ping通`app1`，而`container2`同时属于`isolated_nw`和`local_alias`网络，所以都能ping通

## 多个容器共享一个别名，类似于集群
多个容器共享一个别名，当其中一个容器不可用的时候，将解析到具有同样别名的其他容器，如下面的例子：
1.接这上面的例子，我们再启动一个容器`container4`,并设定别名为`app1`
```bash
$ docker run -tid --network isolated_nw --name container4 --network-alias app1 busybox
```
2.进入container2进行测试,发现这时候ping通的是container4的IP(如果你重复多次ping会发现，app1的IP一会儿是172.19.0.2，一会儿是172.19.0.3)
```bash
/ # ping app1
PING app1 (172.19.0.3): 56 data bytes
64 bytes from 172.19.0.3: seq=0 ttl=64 time=0.064 ms
```
3.停掉container4，再测试，发现，只会ping到container2，而不会再ping到container4
```bash
$ docker stop container4

/ # ping app1
PING app1 (172.19.0.2): 56 data bytes
64 bytes from 172.19.0.2: seq=0 ttl=64 time=0.064 ms
```

## 容器断开网络
使用`docker network disconnect`命令来将容器和网络断开，当容器和网络断开后，容器就不能再和网络中的其他容器进行通信了。
```bash
$ docker network disconnect local_alias container2
```

## 处理过时的网络终端
某些情况下，可能会出现守护进程无法清除过时的连接，这时候我们就需要删除容器，并强行断开它与网络的连接(docker network disconnect -f)，这样之后，你才能成功的连接到网络。

## 删除网络
当所有的容器都已经停止或者都已经断开连接，这时候可以使用`docker network rm`命令来进行删除操作。
