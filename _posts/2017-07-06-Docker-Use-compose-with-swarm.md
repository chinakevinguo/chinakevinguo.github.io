---
layout: post
title: Docker基础-在swarm上使用docker compose
categories: [docker]
description: docker compose with swarm
keywords: docker
---

> 现在看到的是旧版本的独立版的swarm。如果你使用的是docker 1.12或更高的版本，[swarm mode](https://docs.docker.com/engine/swarm/)已经继承到了docker engine中。大多数用户都使用集成的swarm mode。可参考[Getting started with swarm mode](https://docs.docker.com/engine/swarm/swarm-tutorial/)和[Swarm mode CLI commands](https://docs.docker.com/engine/swarm/#swarm-mode-cli-commands)，独立的swarm没有集成到docker engine API和CLI commands中。
<!--more-->
docker compose和docker swarm旨在实现完全集成，这意味着你可以将一个compose应用程序指向一个swarm集群，并使其全部工作，就像使用单个主机一样。

实际的集成程度取决于你使用的[compose file](https://docs.docker.com/compose/compose-file/#versioning)版本:
1.如果你使用版本1以及`links`，你的app可以工作，但是swarm将调度一个主机上的所有容器，因为容器之间的links不能在具有旧网络系统的主机上工作。
2.如果你使用版本2，你的app可以工作，并且不需要任何修改：
  > 受下面描述的[限制](https://docs.docker.com/compose/swarm/#limitations)
  > 只需swarm集群配置使用overlay驱动或自定义网络驱动

参考[Get started with multi-host networking](https://docs.docker.com/engine/userguide/networking/get-started-overlay/)来查看如何使用[Docker Machine](https://docs.docker.com/machine/overview/)和[overlay driver](https://docs.docker.com/engine/userguide/networking/#an-overlay-network-with-docker-engine-swarm-mode)配置swarm 集群.运行后，部署app就变得非常简单：
```bash
$ eval "$(docker-machine env --swarm <name of swarm master machine>)"
$ docker-compose up
```

# 限制

## 构建images
swarm 可以像单台docker主机实例一样从Dockerfile构建image，但是生成的image只会保存在单个的节点上，不会分发到其他节点。
如果你想将服务扩展到多个节点，你需要自己来构建，然后push到registry[例如 docker hub]，然后在`docker-compose.yml`中引用它：
```bash
$ docker build -t myusername/web .
$ docker push myusername/web

$ cat docker-compose.yml
web:
  image: myusername/web

$ docker-compose up -d
$ docker-compose scale web=3
```

## 多依赖

如果service有需要同时调度的依赖，那么很有可能swarm调度的依赖在不同的节点上，是依赖服务无法调度，例如，下面的`foo`同时调度`bar`和`baz`：
```bash
version: "2"
services:
  foo:
    image: foo
    volumes_from: ["bar"]
    network_mode: "service:baz"
  bar:
    image: bar
  baz:
    image: baz
```
那么问题来了，swarm很有可能先在不同的节点上调度`bar`和`baz`（因为它们不依赖其他节点），使得不可能为`foo`选择一个合适的节点，那么应该怎么解决这个问题呢？
要解决上面的问题，我们可以使用手动调度来确保三个service都在同一个节点上，如下：
```bash
version: "2"
services:
  foo:
    image: foo
    volumes_from: ["bar"]
    network_mode: "service:baz"
    environment:
      - "constraint:node==node-1"
  bar:
    image: bar
    environment:
      - "constraint:node==node-1"
  baz:
    image: baz
    environment:
      - "constraint:node==node-1"
```

## 主机端口和重建容器
如果service映射host主机上的端口，如：80:8080，那么在第一次运行`docker-compose up`的时候，可能会报错：
```bash
docker: Error response from daemon: unable to find a node that satisfies
container==6ab2dfe36615ae786ef3fc35d641a260e3ea9663d6e69c5b70ce0ca6cb373c02.
```
产生这个错误主要是因为容器有一个volume(在image或compose文件中定义的)没有映射，所以为了保留数据，compose指示swarm在调度新容器的时候会调度到和旧容器相同的节点，这就导致了端口冲突。

这里有两种方式来解决这个问题：
> 为卷组指定一个名字，并使用volume驱动程序，该驱动程序能将volume安装到容器中，而不管其被调度到什么节点上
如果service仅使用命名volume，则compose不会向swarm发送任何特定的调度命令
```bash
version: "2"

services:
  web:
    build: .
    ports:
      - "80:8000"
    volumes:
      - web-logs:/var/log/web

volumes:
  web-logs:
    driver: custom-volume-driver
```
> 在新建容器之前，删除旧的容器，但是你会丢失卷组里面的数据

```bash
$ docker-compose stop web
$ docker-compose rm -f web
$ docker-compose up web
```

# 容器调度

## 自动调度

某些配置选项将导致在同一swarm节点上自动调度容器，以确保它们正常工作，这些是：
> 版本1中的`network_mode: "service:..." and network_mode: "container:..." (and net: "container:..."`
> `volumes_from`
> `links`

## 手动调度

swarm提供了一组丰富的调度和关联提示，使您可以控制容器所在的位置。他们是通过容器的环境变量来指定的，因此，你可以使用compose的环境变量来设置
```bash
# Schedule containers on a specific node
environment:
  - "constraint:node==node-1"

# Schedule containers on a node that has the 'storage' label set to 'ssd'
environment:
  - "constraint:storage==ssd"

# Schedule containers where the 'redis' image is already pulled
environment:
  - "affinity:image==redis"
```

更多有关swarm调度的内容，可参考[swarm documentation](https://docs.docker.com/swarm/scheduler/filter/)
