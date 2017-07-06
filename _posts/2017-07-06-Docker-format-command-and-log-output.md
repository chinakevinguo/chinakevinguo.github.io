---
layout: post
title: Docker基础-docker 日志输出的格式化
categories: [docker]
description: docker 日志输出的格式化
keywords: docker
---

docker 使用[Go template](https://golang.org/pkg/text/template/)来允许用户操作某些命令和日志的输出格式。详细如下：
* [Docker Images formatting](https://docs.docker.com/engine/reference/commandline/images/posts/#formatting)
* [Docker Inspect formatting](https://docs.docker.com/engine/reference/commandline/inspect/#examples)
* [Docker Log Tag formatting](https://docs.docker.com/engine/admin/logging/log_tags/)
* [Docker Network Inspect formatting](https://docs.docker.com/engine/reference/commandline/network_inspect/)
* [Docker ps formatting](https://docs.docker.com/engine/reference/commandline/ps/#formatting)
* [Docker Volume Inspect formatting](https://docs.docker.com/engine/reference/commandline/volume_inspect/)
* [Docker Version formatting](https://docs.docker.com/engine/reference/commandline/version/#examples)

<!--more-->
# 函数模板
Docker提供了一组基本函数来操作模板元素，下面简单举例几个可用函数的完整列表和示例：
## Json
将结果格式化为json格式
```bash
$ docker inspect --format '{{json .NetworkSettings.IPAddress}}'
```

## upper
将字符串转换成大写
```bash
docker inspect --format '{{upper .NetworkSettings.MacAddress}}' webtest1
```

# lower
将字符转换成小写
```bash
$ docker inspect --format "{{lower .Name}}" container
```

#Title
第一个字母大写
```bash
$ docker inspect --format "{{title .Name}}" container
```

# split
通过指定分隔符，将一个字符串拆分为多个字符串
```bash
$ docker inspect --format '{{split (join .Names "/") "/"}}' container
```

# Join
join 将所有字符串组成一个新的字符串。原始字符串之间使用知道你该的分隔符进行分割。分隔符可以是多个字符组成的字符串
```bash
$ docker ps --format '{{join .Names " or "}}'
```
