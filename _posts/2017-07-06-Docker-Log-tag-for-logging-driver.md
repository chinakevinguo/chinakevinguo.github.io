---
layout: post
title: Docker基础-docker 容器日志命名
categories: [docker]
description: docker 容器内日志命名
keywords: docker
---

tag 选项指定你该为容器的日志如何命名。默认是容器ID的前12个字符。要覆盖默认值，可指定一个tag选项:

```bash
$ docker run --log-driver=fluentd --log-opt fluentd-address=myhost.local:24224 --log-opt tag="mailer"
```
<!--more-->
docker支持一些特殊的标记模板，你可以在指定的时候使用

| 标记 | 描述 |
| :-- | :-- |
| {% raw %}{{.ID}}{% endraw %} | 容器ID的前12个字符 |
| {% raw %}{{.FullID}}{% endraw %} | 容器的全部ID |
| {% raw %}{{.Name}}{% endraw %} | 容器名 |
| {% raw %}{{.ImageID}}{% endraw %} | Image ID 前12个字符 |
| {% raw %}{{.ImageFullID}}{% endraw %} | 全部的Image ID |
| {% raw %}{{.ImageName}}{% endraw %} | Image 名 |
| {% raw %}{{.DaemonName}}{% endraw %}	| docker 进程的名称 |

例如：指定一个 `--log-opt tag="{% raw %}{{.ImageName}}{% endraw %}/{% raw %}{{.Name}}{% endraw %}/{% raw %}{{.ID}}{% endraw %}"`,让其输出到syslog，那么最终他输出的内容如下


```bash
Aug  7 18:33:19 HOSTNAME docker/hello-world/foobar/5790672ab6a0[9103]: Hello from Docker
```

在启动时，系统会在`tag`中设置`container_name` 字段和`{% raw %}{{.name}}{% endraw %}`字段，如果你使用`docker rename`重命名容器，新名称不会反映在日志消息中，这些消息会继续使用原来的容器名。

更高级的用法，可以去参考[go templates](https://golang.org/pkg/text/template/)和[container’s logging context](https://github.com/docker/docker/blob/master/daemon/logger/context.go)

下面是一个syslog的例子，如果我们使用下面的内容,就会得到如下的日志内容

```bash
$ docker run -it --rm \
    --log-driver syslog \
    --log-opt tag="{{ (.ExtraAttributes nil).SOME_ENV_VAR }}" \
    --log-opt env=SOME_ENV_VAR \
    -e SOME_ENV_VAR=logtester.1234 \
    flyinprogrammer/logtester
```

```bash
Apr  1 15:22:17 ip-10-27-39-73 docker/logtester.1234[45499]: + exec app
Apr  1 15:22:17 ip-10-27-39-73 docker/logtester.1234[45499]: 2016-04-01 15:22:17.075416751 +0000 UTC stderr msg: 1
```
