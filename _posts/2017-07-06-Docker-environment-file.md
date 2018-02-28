---
layout: post
title: Docker基础-docker在文件中声明默认环境变量
categories: [docker]
description: docker 在文件中声明默认环境变量
keywords: docker
---

compose 支持在名为`.env`的文件中声明默认环境变量，该文件和`docker-compose.yml`文件在同样的目录 (即docker-compose命令执行的目录)

env文件中的格式是`VAR=VAL`,其中以#开头的为注释，空格也会被忽略。

> **Note:** 运行时环境中存在的值会覆盖`.env`文件中定义的值，同样，通过命令行传递的值也会优先于`.env`中的值

<!--more-->
这些环境变量会用于替换你在compose文件中的环境变量，但也可以定义以下CLI变量：

* COMPOSE_API_VERSION
* COMPOSE_CONVERT_WINDOWS_PATHS
* COMPOSE_FILE
* COMPOSE_HTTP_TIMEOUT
* COMPOSE_TLS_VERSION
* COMPOSE_PROJECT_NAME
* DOCKER_CERT_PATH
* DOCKER_HOST
* DOCKER_TLS_VERIFY

# 更多compose文档

* [User guide](https://docs.docker.com/compose/)
* [Command line reference](https://docs.docker.com/compose/reference/)
* [Compose file reference](https://docs.docker.com/compose/compose-file/)
