---
layout: post
title: Docker基础-docker 环境变量在compose文件中的使用
categories: [docker]
description: docker 环境变量在compose文件中的使用
keywords: docker
---

# 在compose文件中引用环境变量

可以在compose文件中引用运行`docker-compose`所在的shell中的环境变量，如：

```bash
web:
  image: "webapp:${TAG}"
```
<!--more-->
# 在容器中设置环境变量

可以在compose文件中的`environment`关键字下设置容器的环境变量，就像使用`docker run -e VARIABLE=VALUE`一样

```bash
web:
  environment:
    - DEBUG=1
```

# 将环境变量传递到容器

可以在compose文件中的`environment`关键字下定义一个环境变量而不是直接赋值，就像是`docker run -e VARIABLE`

```bash
web:
  environment:
    - DEBUG
```

# env_file配置选项

可以使用compose文件中的`env_file`选项从一个外部文件传递多个环境变量到容器中，就像`docker run --env-file=FILE`

```bash
web:
  env_file:
    - web-variables.env
```

# 使用docker-compose run设置环境变量

就像`docker run -e`,可以使用`docker-compose run -e`为一次性容器设置环境变量

```bash
docker-compose run -e DEBUG=1 web python console.py
```
也可以不赋值，直接从shell变量中取值
```bash
docker-compose run -e DEBUG web python console.py
```
DEBUG的值是从执行compose文件所在的shell的同一个环境变量取得。

# .env文件

可以在环境文件`.env`设置默认的环境变量，这些环境变量可以在compose文件中调用
```bash
$ cat .env
TAG=v1.5

$ cat docker-compose.yml
version: '2.0'
services:
  web:
    image: "webapp:${TAG}"
```

当执行`docker-compose up`命令时，上面定义的web服务将使用webapp:v1.5镜像，可以使用config命令来打印出来：
```bash
$ docker-compose config
version: '2.0'
services:
  web:
    image: 'webapp:v1.5'
```

在shell中的环境变量将比定义在`.env`文件中的环境变量优先。如果在shell中设置一个不同的TAG，镜像将优先使用shell中的定义，而不是.evn文件中的：
```bash
$ export TAG=v2.0
$ docker-compose config
version: '2.0'
services:
  web:
    image: 'webapp:v2.0'
```

# 使用环境变量来配置compose

某些环境变量可用来配置以改变`docker compose`的命令行特性，以`COMPOSE_`或`DOCKER_`开头，详细信息参考[CLI Environment Variables](https://docs.docker.com/compose/reference/envvars/)
