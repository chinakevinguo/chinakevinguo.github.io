---
layout: post
title: Docker基础-docker compose简介
categories: [docker]
description: docker compose简介
keywords: docker
---

Compose 是一个定义和运行多个docker容器的应用程序的工具。你可以使用compose file来配置你的应用服务。然后，使用单个命令，docker就会根据你的compose配置文件来创建和启动所有的服务。想要了解更多关于compose的特性，可以参考后面的[the list of features](https://docs.docker.com/compose/overview/#features)

compose 非常适用于开发，测试和临时环境以及CI工作流。我们可以在下面的[Common Use Cases](https://docs.docker.com/compose/overview/#common-use-cases)中了解更多。
<!--more-->
使用compose一般有三步：

1.在`Dockerfile`中定义好你的应用的运行环境
2.在`docker-compose.yml`中定义组成应用的服务，以便他们可以在一个隔离的环境中一起运行
3.运行`docker-compose up`命令，compose将会启动和运行你定义的应用服务

`docker-compose.yml`文件内容如下：
```yaml
version: '2'
services:
  web:
    build: .
    ports:
    - "5000:5000"
    volumes:
    - .:/code
    - logvolume01:/var/log
    links:
    - redis
  redis:
    image: redis
volumes:
  logvolume01: {}
```

更多关于compose文件的信息，可参考[Compose file reference](https://docs.docker.com/compose/compose-file/)

compose具有管理应用程序整个生命周期的命令：

* start,stop和rebuild 服务
* 查看正在运行服务的状态
* 运行服务的日志的输出
* 在服务上运行一次性命令

# compose 文档

* [installing compose]()
* [getting started]()
* [get started with django]()
* [get start with rails]()
* [get started with wordpress]()
* [frequently asked questions]()
* [command line reference]()
* [compose file reference]()

# 特性

## 单个主机上的多个环境的隔离

compose 使用项目名称来互相隔离环境。你可以在几个不同的上下文中使用此项目的名称：

* 在dev主机上，创建单个环境的多个副本(例如，您要为项目的每个功能部分运行稳定的副本)
* 在CI服务器上，为了使构建不会互相干扰，可以将项目名称设定为唯一的内部版本号
* 在共享主机或dev主机上，以防止可能使用相同服务名称的不同项目互相干扰

默认的项目名称是项目 目录的基本名称。你可以使用[`-p`命令行选项](https://docs.docker.com/compose/reference/overview/)或[`COMPOSE_PROJECT_NAME`环境变量](https://docs.docker.com/compose/reference/envvars/#compose-project-name)设置自定义项目名称。

## 当容器创建时，保留volume 数据

compose 会保留你的服务所使用的所有的vomules。当`docker-compose up`运行时，如果找到先前运行的任何容器，都会将volume从旧容器复制到新容器。以确保你在volume中的数据不会丢失。

如果是在windows上运行`docker-compose`，需要根据特定的环境设定必要的环境变量。

## 只重建发生改变的容器

compose 缓存配置来创建容器。当你在没有发生改变时重启服务。compose会重新使用已经存在的容器。重新使用容器，以为着你可以快速的改变你的环境

## 变量和组合在环境之间的移动

compose 支持在compose file中使用变量。你可以使用这些变量来为不同的环境或不同的用户自定义组合。详情[variable substitution](https://docs.docker.com/compose/compose-file/#variable-substitution)

你还可以使用`extend`字段或通过创建多个compose文件来扩展compose文件。详情参考[extends](https://docs.docker.com/compose/extends/)

# compose 使用例子

compose 可以以很多不同的方式使用，一些常用的使用例子如下：

## 开发环境

当你开发软件时，在独立的环境中运行应用程序并与其交互十分重要，compose命令行工具可用于创建环境并与之交互。
compose 文件提供了一种记录和配置所有应用服务之间依赖(数据库，队列，缓存，web服务API等)的方法。使用compose命令行工具，你可以使用单个命令`docker-compose up`为每个依赖关系创建一个或多个容器。

总之，这些功能为开发人员提供了一个方便的方法来开始项目。compose可以将多页的"开发人员入门指南"减少到单个机器可读的compose文件和几个命令。

## 自动测试环境

持续开发或持续集成中一个重要的部分是自动化测试套件。自动化端到端测试需要一个运行测试的环境。compose提供了一种方便的方法来创建和销毁隔离测试环境的测试套件。通过在compose文件中定义完整的环境，你只需要几个命令就可以创建和销毁这些环境：
```bash
$ docker-compose up -d
$ ./run_tests
$ docker-compose down
```

## 单台主机的部署

compose 在以前一直专注于开发和测试，但随着版本的更新，我们现在正在开发更多面向生产的功能。你可以使用compose来部署到远程的docker engine。docker engine可以是使用docker machine或docker swarm集群提供的单个实例。

# Getting help

docker compose目前正在积极开发中，如果，你需要获取帮助，想要贡献，或者只是想要和志同道合的人沟通，我们有以下开发的渠道：
* 反馈bug或者功能请求，[issue tracker on Github](https://github.com/docker/compose/issues)
* 和其他人讨论，可以加入IRC上的`docker-compose`频道
* 提交贡献，[pull request on Github](https://github.com/docker/compose/pulls)
