---
layout: post
title: Docker基础-docker multi-host网络之overlay
categories: [docker]
description: docker multi-host网络之overlay
keywords: docker
---
本章主要是通过例子来讲解multi-host network，Docker engine 支持使用`overlay`驱动的网络开箱即用，不像`bridge`网络，`overlay`网络的使用，需要做一些前提准备工作：
* [Docker Engine在swarm模式下运行](https://docs.docker.com/engine/userguide/networking/get-started-overlay/#overlay-networking-and-swarm-mode)
OR
* [Docker Engine运行在使用kv存储的集群](https://docs.docker.com/engine/userguide/networking/get-started-overlay/#overlay-networking-with-an-external-key-value-store)

<!--more-->
## overlay networking and swarm mode
在[swarm mode](https://docs.docker.com/engine/swarm/swarm-mode/)下运行Docker Engine，你可以在管理节点上创建`overlay`网络
swarm使`overlay`网络只有在swarm中的service需要时才可用，比如当你创建了一个`overlay`网络时，管理节点会首先将`overlay`同步到其他管理节点，然后当你创建的service使用这个`overlay`网络的时候，`overlay`才会同步到这些service所在的节点。
下面的例子显示了如何在管理节点上创建网络，并将其应用到swarm中的service：
```bash
# create an overlay network `my-multi-host-network`
$ docker network create --driver overlay --subnet 10.0.9.0/24 my-multi-host-network

# show overlay network ,可以看到它的SCOPE是swarm
$ docker network ls |grep my-multi-host-network
3cy4vcgiin5g        my-multi-host-network   overlay             swarm

# create an nginx service and extend the my-multi-host-network to nodes where the service's run
$ docker service create --network my-multi-host-network --name my-web --replicas 5 nginx
```
`overlay`网络不会覆盖到那些没有被swarm管理起来的容器，更多信息可以参考[Docker swarm mode overlay network security model](https://docs.docker.com/engine/userguide/networking/overlay-security-model/)
也可以参考[Attach service to an overlay network](https://docs.docker.com/engine/swarm/networking/)

## overlay networking with an external key-value store
如果使用外部的key-value存储，你需要以下几步：
* 一个可以访问的key-value存储.docker  支持`Consul`,`Etcd`,`ZooKeeper`等key-value存储。
* 一个连接到key-value存储的集群
* 在集群中的每台机器上有正常运行的Docker Engine daemon
* 集群中的主机必须拥有唯一的主机名，因为key-value存储使用主机名来标识集群成员

`Docker machine`和`Docker swarm`不强制使用key-value存储来体验multi-host network，这里用key-value 存储主要是为了展示他们是怎么集成的。这个例子中，你将使用docker machine来创建key-value store 和集群。
> **Note:** 在swarm模式下运行的Docker engine和使用外部存储的网络不兼容

### 前提条件
开始之前，请确认你的网络上安装了最新版的Docker Engine 和Docker Machine.这个例子中使用的是Virtualbox，如果你在Windows或MAC上安装了Docker Toolbox，那么这些东西，你已经安装了。
如果之前你已经在windows上安装了virtualbox，那么这时你只需要按照[Install Docker Machine](https://docs.docker.com/machine/install-machine/)安装docker-machine即可

### 配置key-value store
`overlay`网络依赖key-value存储.key-value存储保存着一些关于网络的状态信息，包括，发现，网络，终端，IP地址等等.Docker 支持Consul,Etcd,ZooKepper等key-value存储，这里使用的是Consul.
1.创建一个名叫`mh-keystore`的虚拟机.
```bash
$ docker-machine create -d virtualbox mh-keystore
```
当你创建这个虚拟机时，会自动在虚拟机中创建Docker Engine.相对于手动安装Consul，用这种方式可以直接从Docker Hub上拉去Consul镜像
2.将`mh-keystore`机器的环境变量添加到本地
```bash
# 下面这条命令的意思将`docker-machine env mh-keystore`输出的结果作为命令在本地执行
$ eval "$(docker-machine env mh-keystore)"
```
3.在`mh-keystore`虚拟机里面运行一个consul容器
```bash
$  docker run -d \
     -p "8500:8500" \
     -h "consul" \
     progrium/consul -server -bootstrap
```
4.我们可以看到在`mh-keystore`里面已经有了一个运行着consul的容器,并监听到8500端口
```bash
docker@mh-keystore:~$ docker ps
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS                                                                            NAMES
c7075a337757        progrium/consul     "/bin/start -server -"   11 hours ago        Up About an hour    53/tcp, 53/udp, 8300-8302/tcp, 8400/tcp, 8301-8302/udp, 0.0.0.0:8500->8500/tcp   drunk_lovelace
```

### 创建swarm集群
在这步，使用`docker-machine`来为你的网络创建主机，此时，你不会真正的创建网络，你会在virtualbox中创建几个虚拟机，其中一个作为swarm master，master要首先创建，然后当你创建其它节点时，`overlay`网络，覆盖到所需的节点上。
1.创建swarm master
```bash
$ docker-machine create \
 -d virtualbox \
 --swarm --swarm-master \
 --swarm-discovery="consul://$(docker-machine ip mh-keystore):8500" \
 --engine-opt="cluster-store=consul://$(docker-machine ip mh-keystore):8500" \
 --engine-opt="cluster-advertise=eth1:2376" \
 mhs-demo0
```
创建可能需要几分钟，这里要说到几个参数：
* --swarm 将机器加入到swarm集群中
* --swarm-master 将机器配置成swarm master
* --swarm-discovery 在使用swarm集群的时候使用discovery服务
* --engine-opt 指定在创建machine时的参数，比如这里指定了保存`overlay`网络的存储

2.创建另外一台主机，并加入到swarm集群
```bash
$ docker-machine create -d virtualbox \
    --swarm \
    --swarm-discovery="consul://$(docker-machine ip mh-keystore):8500" \
    --engine-opt="cluster-store=consul://$(docker-machine ip mh-keystore):8500" \
    --engine-opt="cluster-advertise=eth1:2376" \
  mhs-demo1
```
3.列出所有的机器，并确认他们都已经在运行
```bash
$ docker-machine ls
NAME          ACTIVE   DRIVER       STATE     URL                         SWARM                DOCKER    ERRORS
mh-keystore   -        virtualbox   Running   tcp://192.168.99.100:2376                        v1.12.5
mhs-demo0     *        virtualbox   Running   tcp://192.168.99.102:2376   mhs-demo0 (master)   v1.12.5
mhs-demo1     -        virtualbox   Running   tcp://192.168.99.103:2376   mhs-demo0            v1.12.5
```
此时，你已经拥有了一组在网络上运行的主机，可以开始为这些主机的容器创建multi-host-network。

### 创建Overlay网络
1.将swarm master的环境变量添加到本地
```bash
$ eval $(docker-machine env --swarm mhs-demo0)
```
2.进入`mhs-demo0`，查看`docker info`信息
```bash
$ docker-machine ssh mhs-demo0

$ docker info
Containers: 3
 Running: 3
 Paused: 0
 Stopped: 0
Images: 2
Server Version: 1.12.5
Storage Driver: aufs
 Root Dir: /mnt/sda1/var/lib/docker/aufs
 Backing Filesystem: extfs
 Dirs: 12
 Dirperm1 Supported: true
Logging Driver: json-file
Cgroup Driver: cgroupfs
Plugins:
 Volume: local
 Network: overlay bridge null host
Swarm: inactive
Runtimes: runc
Default Runtime: runc
Security Options: seccomp
Kernel Version: 4.4.39-boot2docker
Operating System: Boot2Docker 1.12.5 (TCL 7.2); HEAD : fc49b1e - Fri Dec 16 12:44:49 UTC 2016
OSType: linux
Architecture: x86_64
CPUs: 1
Total Memory: 995.8 MiB
Name: mhs-demo0
ID: 7IIW:YM6F:ONBD:34ID:7P5A:IWMR:R5EU:IKPG:RQ7U:GFNY:QQ6J:F5GN
Docker Root Dir: /mnt/sda1/var/lib/docker
Debug Mode (client): false
Debug Mode (server): true
 File Descriptors: 44
 Goroutines: 73
 System Time: 2016-12-31T05:48:48.647020582Z
 EventsListeners: 1
Registry: https://index.docker.io/v1/
Labels:
 provider=virtualbox
Cluster Store: consul://192.168.99.100:8500
Cluster Advertise: 192.168.99.102:2376
Insecure Registries:
 127.0.0.0/8
```
3.开始创建`overlay`网络
```bash
$  docker network create --driver overlay --subnet=10.0.9.0/24 my-net
```
你只需要在swarm master上创建`overlay`网络，后续使用的时候，swarm会自动同步到其他机器上
> **NOTE:** 强烈建议在创建网络的时候指定`--subnet`选项，如果不指定的话，可能会和其他没有在swarm集群中管理的机器IP冲突

4.分别进入`mhs-demo0`和`mhs-demo1`容器，查看刚刚创建的`my-net`网络是否生效
```bash
# 进入mhs-demo0
$ docker-machine ssh mhs-demo0

docker@mhs-demo0:~$ docker network ls
NETWORK ID          NAME                DRIVER              SCOPE
9b5899502854        bridge              bridge              local
2b0344b0f275        docker_gwbridge     bridge              local
9bdb81aab8b8        host                host                local
89645f6295f4        my-net              overlay             global
71113ce8e823        none                null                local

#进入mhs-demo1
$ docker-machine ssh mhs-demo1

docker@mhs-demo1:~$ docker network ls
NETWORK ID          NAME                DRIVER              SCOPE
4ad6d85d6654        bridge              bridge              local
037c69787c0c        docker_gwbridge     bridge              local
b823a92977d6        host                host                local
89645f6295f4        my-net              overlay             global
700571c2dae2        none                null                local
```
可以看到`my-net`网络在两个主机上都已经生效了

### 测试.在`my-net`网络上运行应用

1.我们在`mhs-demo0`上运行一个nginx web应用
```bash
$ docker run -itd --name=web --network=my-net --env="constraint:node==mhs-demo0" nginx
```
2.在`mhs-demo1`上运行一个busybox容器，并获取Nginx服务器主页上的内容
```bash
docker@mhs-demo1:~$ docker run -it --rm --network=my-net --env="constraint:node==mhs-demo1" busybox wget -O- http://web
Connecting to web (10.0.9.2:80)
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
-                    100% |*******************************|   612   0:00:00 ETA

```

### 检查外部网络连接
如你所见，Docker内置的`overlay`网络在使用同样网络的同台主机的容器之间是开箱即用的。此外，容器连接到多台主机时会自动连接到`docker_gwbridge`网络，这个网络会让容器拥有集群外的连接。

我们来查看nginx容器的网络接口信息
```bash
$ docker exec web ip addr
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
13: eth0@if14: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UP group default
    link/ether 02:42:0a:00:09:02 brd ff:ff:ff:ff:ff:ff
    inet 10.0.9.2/24 scope global eth0
       valid_lft forever preferred_lft forever
    inet6 fe80::42:aff:fe00:902/64 scope link
       valid_lft forever preferred_lft forever
16: eth1@if17: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default
    link/ether 02:42:ac:12:00:02 brd ff:ff:ff:ff:ff:ff
    inet 172.18.0.2/16 scope global eth1
       valid_lft forever preferred_lft forever
    inet6 fe80::42:acff:fe12:2/64 scope link
       valid_lft forever preferred_lft forever
```
我们可以看到，`eth0`连接到`my-net`网络,`eth1`连接到`docker_gwbridge`网络
