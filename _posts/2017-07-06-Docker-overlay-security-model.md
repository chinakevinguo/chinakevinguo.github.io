---
layout: post
title: Docker基础-docker muti-host之overlay在docker swarm安全模式下的使用
categories: [docker]
description: docker muti-host之overlay在docker swarm安全模式下的使用
keywords: docker
---

# Docker swarm mode overlay network security model
overlay 网络在docker swarm下的安全模式也是开箱即用的。swarm 节点之间通过`gossip`协议来交换overlay 网络信息。节点使用GCM模式下的[AES algorithm](https://en.wikipedia.org/wiki/Galois/Counter_Mode)对通过`gossip`协议来交换的信息进行加密和认证.管理节点每12小时轮询加密key一次

<!--more-->

你还可以加密在overlay网路上运行的容器之间的交换数据，你可以通过使用`--opt`来开启加密：
```bash
$ docker network create --opt encrypted --driver overlay my-multi-host-network
```

当你开启了overlay 加密，docker 会为所有依附overlay网路的服务节点创建一个IPSEC通道，这个IPSEC通道也是使用GCM模式下的AES算法进行加密，并且每12小时轮询一次。

# Swarm mode overlay networks and unmanaged containers
因为在swarm 模式下的overlay 网络使用来自管理节点的加密密钥来加密gossip通信，只有作为swarm 中的任务运行的容器才能访问密钥，所以在swarm之外的容器是无法附加到overlay网络的。
```bash
$ docker run --network my-multi-host-network nginx

docker: Error response from daemon: swarm-scoped network
(my-multi-host-network) is not compatible with `docker create` or `docker
run`. This network can only be used by a docker service.
```

要解决这类问题，你可以将没在集群的容器，迁移到集群即可：
```bash
$ docker service create --network my-multi-host-network my-image
```

因为swarm mode是一个可选功能，docker engine保留了其向后的兼容性，你可以继续key-value的方式来存储overlay网络，但是，我们建议使用集群模式，不仅仅是因为前面所说的安全，swarm模式下还提供了很多API来提供更大的可伸缩性。
