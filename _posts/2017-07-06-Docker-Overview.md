---
layout: post
title: Docker基础-docker 简介
categories: [docker]
description: docker 简介
keywords: docker
---
# 什么是Docker
Docker是一个提供开发、传输及运行应用程序的开放平台，Docker能将应用程序和基础架构平台进行分离，以便快速交付，使用Docker，你能用像管理基础架构一样的的方式来管理你的应用程序，通过利用Docker来开发，测试，部署，传输代码，可以显著的提高开发部署的效率。
Docker使用Google公司推出的Go语言进行开发实现，基于Linux内核的[Control groups](https://zh.wikipedia.org/wiki/Cgroups),[Union file systems](https://en.wikipedia.org/wiki/Union_mount)，[Namespace](https://en.wikipedia.org/wiki/Linux_namespaces)等技术，对进程进行封装隔离，属于操作系统层面的虚拟化技术。由于隔离的进程独立于宿主机和其他被隔离的进程，因此也称其为容器。最初实现是基于LXC，从0.7之后开始去除LXC，转而使用自行开发的libcontainer，从1.11开始，则进一步演进为使用RUNC和containerd
<!--more-->
# Docker Engine
Docker engine 是一个C/S架构应用程序，包含如下组件，如下图：
* Docker server daemon
一个长时间运行的守护进程
* REST API
程序和damon通信的接口
* Docker client CLI
命令行界面
这里的Docker client CLI就是通过REST API 和Docker server daemon进行通信，而其它的底层程序则是通过底层的API来和REST API接口进行通信，最终和Docker server daemon进行通信
![](/images/engine-components-flow.png)

# Docker VS Virtual Machines
下面的图片比较了Docker和传统虚拟化方式的不同之处。传统虚拟机技术是虚拟出一套硬件后，在其上运行一套完整的操作系统，再在系统上运行应用程序；而Docker的应用程序则是直接运行于主机的内核之上，容器没有自己的内核，也没有进行硬件虚拟化。因此容器比传统虚拟机更为轻量。
![Virtual Machines](/images/virtualization.png)
![Docker](/images/docker.png)

*The underlying technology*
* Namespaces
Docker 使用namespaces来对容器进行隔离，当你运行一个容器时，Docker为该容器创建一组namespaces，这个namespaces提供一个隔离层，容器的所有操作都被限制在这个单独的namespaces中
Docker在Linux上使用如下的namespaces：
> * PID 进程隔离
> * NET 网络隔离
> * IPC 消息队列隔离
> * MNT 文件系统挂载隔离
> * UTS 主机名和域名隔离
> * USER 用户和用户组隔离
* Control groups
Cgroup技术将应用程序限制为特定的一进程集，Cgroup允许Docker Engine将可用的硬件资源共享到容器，并且可以对其进行限制和约束，例如，你可以限制容器的内存大小
* Union file systems
UFS，其实就是一个层级概念，将多个目录挂载到同一个目录下，然后进行修改，修改的内容会被重新保存到另外的层里面，而不会影响到原有挂载层数据
* Container format
Docker将namespaces，cgroup和UnionFS组合在一起，形成了一个libcontainer

# Docker的存储方式-Storage Driver
为了能有效的理解Docker的存储方式，首先，你必须要了解如何构建和存储镜像，然后，了解如何通过镜像使用容器，最后，了解如何使用镜像和容器。
在这之前，你必须先了解layers和image，container的概念.
Docker模型的核心部分是有效利用分层镜像机制，镜像可以通过分层来进行继承，基于基础镜像，可以制作各种具体的应用镜像，不同的Docker容器就可以共享一些基础的文件系统层，同时加上自己独有的改动层，大大提高了存储的效率，其中主要的机制就是分层模型和ufs。针对镜像存储，docker采用了集中不同的存储drivers，包括：aufs，devicemapper，btrfs和overlay
# 内容寻址存储
在Docker 1.10版中引入了一个新的内容寻址存储模型，这是一种在磁盘上定位镜像层和数据层的新方式。原来的版本中，镜像层和数据层使用随机生成的UUID来进行存储，这样可能会出现ID冲突，而新版本的存储模型，使用哈希来进行存储，提高了安全性，同时也保证了数据的完整性。
另：
镜像是由多个只读层组成，并且多个只读层是可以被多个镜像共享的，如果其中一个数据层，在某个镜像中已经存在，那么根据内容寻址存储的原理，该数据层，就不需要被再此下载，可以节省空间。
容器是建立在镜像层之上的一个可读可写的层，多个容器层可以共享一个镜像层，容器层内数据的修改是遵循cow原则，更高效。
![container-layers-cas](/images/container-layers-cas.jpg)
可以看到所有的镜像层ID都是加密的哈希值，但是容器层依旧是使用的随机生成的UUID
# copy-on-write
写时复制，多个进程共享资源，只有当某个进程需要进行修改的时候，才会复制一份给该进程，而其他进程依然使用原始的数据。
# Images and layers
Images其实是由多个表示不同文件系统的只读层组成，层级从底层开始，逐层堆叠，形成一个完整的Image,Docker storage driver就负责来堆叠这些层
![image-layers](/images/image-layers.jpg)
# Container and layers
当你创建了一个新的容器，你就是在Images layers的最上层添加了一个新的可读，可写的层，这个层就称之为容器，所有的变动都在这个容器层里面发生。
![container-layers](/images/container-layers.jpg)
