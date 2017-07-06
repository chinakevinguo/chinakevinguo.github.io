---
layout: post
title: Docker基础-docker stacks
categories: [docker]
description: docker stacks
keywords: docker
---

> 该章节介绍的功能都是实验性的，在变成正式版本之前，可能发生变动

> 这些内容来自[docker/docker repo](https://github.com/docker/docker)的[Docker Stacks and Distributed Application Bundles](https://github.com/docker/docker/blob/v1.12.0-rc4/experimental/docker-stacks-and-bundles.md)

# 概述
docker stacks 和分布式应用程序bundles是在docker 1.12和docker compose 1.8中引入的一个新特性，目前还在实验阶段，同Engine API中的swarm、nodes和service同时进行

截至目前，大家已经可以选定一个Dockerfile，并利用docker build命令由此创建镜像。使用docker run命令则可启动容器。这条命令亦能够轻松同时启动多套容器。另外，大家也可以使用Docker Compose文件并利用docker-compose scale命令对容器进行规模扩展。
![](/images/posts/Dockerfile.jpg)
那么同样，一个docker-compose.yml可以创建一个分布式的应用程序bundle。使用docker-compose bundle可以启动docker stack。其专门面向多套容器的迁移需求，每个捆绑包都可以作为stack在运行时进行部署。
![](/images/posts/Docker-compose.jpg)

简单来说，我们可以使用一个类比来进行说明：
Dockerfile --> 镜像 --> 容器
Docker Compose --> 分布式应用程序包 --> Docker Stack

从Docker 1.12和compose 1.8开始，这些功能是实验性的。Docker Engine和Docker Registry 都不支持捆绑包的分发。

<!--more-->
# 生成一个捆绑包
生成捆绑包最简单的方法就是使用docker-compose从docker-compose.yml中生成。当然，这仅仅是一种可能的方式，用同样的方式， docker build不是唯一能生成docker image的方式。

1.新建docker-compose.yml文件
```bash
$ cat docker-compose.yml
version: "2"
services:
 db:
   container_name: "db"
   image: arungupta/oreilly-couchbase:latest
   ports:
     - 8091:8091
     - 8092:8092
     - 8093:8093
     - 11210:11210
 web:
   image: arungupta/oreilly-wildfly:latest
   depends_on:
     - db
   environment:
     - COUCHBASE_URI=db
   ports:
     - 8080:8080
```

2.拉取服务镜像
```bash
$ docker-compose pull db
$ docker-compose pull web
```

3.生成捆绑包
```bash
$ docker-compose bundle
WARNING: Unsupported key 'depends_on' in services.web - ignoring
WARNING: Unsupported key 'container_name' in services.db - ignoring
Wrote bundle to composetest.dab
```

# 从bundle创建一个stack

> **Note:** 因为目前stack和应用程序捆绑包还在试验阶段，所以最好使用最新版的docker-engine才会有这些新特性，想要使用这些新特性，可参考[experimental build README](https://github.com/docker/docker/blob/master/experimental/README.md)

在这之前，我们先开启这些实验版的特性
```bash
$ vi /lib/systemd/system/docker.service
  ExecStart=/usr/bin/dockerd --experimental

# 重启服务
$ systemctl daemon-reload
$ systemctl restart docker

# 确认是否开启成功
$ docker version -f '{ { .Server.Experimental } }'
true
```

使用`docker deploy`命令来创建stack
```bash
$ docker deploy --help

Usage:	docker deploy [OPTIONS] STACK

Deploy a new stack or update an existing stack

Options:
      --bundle-file string    Path to a Distributed Application Bundle file
  -c, --compose-file string   Path to a Compose file
      --help                  Print usage
      --with-registry-auth    Send registry authentication details to Swarm agents
```

依据前面生成的捆绑包来部署stack
```bash
$ docker deploy --bundle-file composetest.dab composetest
Loading bundle from composetest.dab
Creating network composetest_default
Creating service composetest_db
Creating service composetest_web
```

检查我们刚部署的服务
```bash
$ docker service ls
ID            NAME             MODE        REPLICAS  IMAGE
1658d7kh5gs9  composetest_web  replicated  1/1       arungupta/oreilly-wildfly@sha256:d567ade7bb82ba8f15a85df0c6d692d85c15ec5a78d8826dfba92756babcb914
pr6ympnpgv9g  composetest_db   replicated  1/1       arungupta/oreilly-couchbase@sha256:f150fcb9fca5392075c96f1baffc7f893858ba763f3c05cf0908ef2613cbf34c
```

# 管理stacks
使用`docker stack`命令来进行管理
```bash
$ docker stack --help

Usage:	docker stack COMMAND

Manage Docker stacks

Options:
      --help   Print usage

Commands:
  deploy      Deploy a new stack or update an existing stack
  ls          List stacks
  ps          List the tasks in the stack
  rm          Remove the stack
  services    List the services in the stack

Run 'docker stack COMMAND --help' for more information on a command.
```

# Bundle 文件格式
分布式应用程序包用`JSON`格式来描述。当bundles以文件方式保存时，文件的后缀是`.dab`。
一个bundle有两个顶级字段：`version`和`services`。docker1.12工具使用的是0.1版

`services`是指构成应用程序的服务。他们对应1.12 Docker Engine API中引述的新services对象

一个`services`包含如下字段：
**Image[必须]**
服务将运行的image。docker images应该使用hash码来引用，如：`arungupta/oreilly-couchbase@sha256:f150fcb9fca5392075c96f1baffc7f893858ba763f3c05cf0908ef2613cbf34c`
**Command**[]
在服务容器中运行的命令
**Args**[]
传递给服务容器的参数
**Env**[]
环境变量
**Labels**map[string]
设置标签
**Ports**[]
服务端口(由端口和协议组成)，服务描述只能指定要暴露的容器端口，具体要隐射到主机的那些端口，就有管理员自己决定了
**WorkingDir**
服务容器内部的工作目录
**User**
用户名或UID
**Networks[]**
应该要连接到服务容器的网络

> **Note:** 目前尚有很多选项不支持DAB格式，包括volume mount
