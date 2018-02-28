---
layout: post
title: Docker基础-docker compose网络
categories: [docker]
description: docker compose 网络
keywords: docker
---

> **注意：** 本文涉及的compose只适用于compose文件格式为version 2的版本，v1(旧的)不支持网络功能

默认compose会为你的app配置一个单独的网络。服务中的每个容器加入到这个默认的网络且在这个网络的容器都能互相通信，它们也能通过与容器名称相同的主机名发现对方。

> **注意：** app的网络基于"项目名称"设置网络名称，这个项目名称基于项目所处的目录名。可以使用`--project-name`选项或`COMPOSE_PROJECT_NAME`环境变量来覆盖。

<!--more-->
例如，假设app在一个名为myapp的目录，`docker-compose.yml`内容如下：
```bash
version: '2'

services:
  web:
    build: .
    ports:
      - "8000:8000"
  db:
    image: postgres
```
当执行`docker-compose up`时，网络部分设置如下：
1.创建了称之为myapp_default的网络
2.使用web的配置创建容器，然后这个容器加入到myapp_default网络
3.使用db的配置创建容器，这个容器加入到myapp_default网络

每个容器现在能直接查找主机名web或db来得到容器的IP地址。例如，web应用程序的代码可以连接URL postgres://db:5432并开始使用postgres数据库。
由于web明确的映射了一个端口，外部网络也就能通过在docker主机的网络接口的8000端口连接容器。

# 容器更新

如果更改了服务的配置并执行`docker-compose up`来更新它，将删除旧的容器并且新的容器会加入到相同的网络，分配到了不同的IP地址，不过主机名不变。运行中的容器应该能够查找主机名并连接到新的地址，不过旧的地址将失效。

如果任何一个容器与旧容器有连接，它们会被关闭掉。容器有责任检测这种情况然后重新查找旧容器的主机来重新连接。

# Links

links可以为一个容器定义一个额外的别名。即使服务没有启动，它们也能进行通信。默认任何服务都可以通过该服务的名称访问其他的服务。例如：在web容器中可以通过db和database访问db容器

```bash
version: '2'
services:
  web:
    build: .
    links:
      - "db:database"
  db:
    image: postgres
```

# 多主机网络

当在swarm集群中部署compose app的时候，你可以使用内置的`overlay`driver来启用容器之间的多主机通信，这不会更改你的compose文件和app code。

请参阅[Getting started with multi-host networking](https://docs.docker.com/engine/userguide/networking/get-started-overlay/)来了解怎么配置swarm集群。swarm默认是使用`overlay`网络驱动，当然你也可以自己自定义。详情在下一段

# 指定自定义网络

除了使用默认的app网络之外，还可以使用最顶层的`networks`关键字来指定自定义的网络。这让你可以创建更复杂的网络并制定自定义网络驱动及其选项。也可以使用它将服务连接到不是有compose管理的外部网络。

每个服务都能指定由networks关键字配置的网络，可以配置service级别和top级的网络。

下面的示例compose文件定义了两个自定义网络。proxy服务与db服务隔离，因为它们没有指定相同的网络。
```bash
version: '2'

services:
  proxy:
    build: ./proxy
    networks:
      - front
  app:
    build: ./app
    networks:
      - front
      - back
  db:
    image: postgres
    networks:
      - back

networks:
  front:
    # Use a custom driver
    driver: custom-driver-1
  back:
    # Use a custom driver which takes special options
    driver: custom-driver-2
    driver_opts:
      foo: "1"
      bar: "2"
```

# 配置默认网络

除了指定你自己的网络之外，还可以通过在名为default的网络下定义一个条目来更改应用范围内的默认网络设置：
```bash
version: '2'

services:
  web:
    build: .
    ports:
      - "8000:8000"
  db:
    image: postgres

networks:
  default:
    # Use a custom driver
    driver: custom-driver-1
```

# 使用预先存在的网络

如果你希望你的容器加入一个预先存在的网络，使用external选项：
```bash
networks:
  default:
    external:
      name: my-pre-existing-network
```
compose检测到有external选项后，不会创建名为[PROJECTNAME]_default的网络，而是会查找一个名为my-pre-existing-network的网络，并将应用程序连接到它。
