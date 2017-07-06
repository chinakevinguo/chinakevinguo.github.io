---
layout: post
title: Docker基础-自动启动容器
categories: [docker]
description: 自动启动容器
keywords: docker
---

# 自动启动容器

从docker1.2开始，[重启策略](https://docs.docker.com/engine/reference/run/#restart-policies-restart)内置在docker中，用于在容器退出时重启容器。如果设置了重启策略，它将在docker daemon启动的时候启用，通常发生在系统启动之后。重启策略将确保链接容器按正确的顺序启动。

如果重启策略不适合你的环境，你可以使用像[upstart](http://upstart.ubuntu.com/),[systemd](http://freedesktop.org/wiki/Software/systemd/) or [supervisor](http://supervisord.org/)代替

<!--more-->
# 使用进程管理器

docker 默认不会设定任何重启策略，因为它们与大多数进程管理器冲突，所以如果你正在使用进程管理器，请不要设置新的重启策略。

当你已经完成了你的镜像设定，并且已经开心的运行一个容器之后，你可以附加一个进程管理器来管理它。当你运行`docker start -a`,docker会自动附加到正在运行的容器，或者，如果需要启动它，并转发所有信号，一边进程管理器可以检测到容器何时停止并正确重新启动它。

下面有一些用于`systemd`的例子：

# systemd
在这个例子中，我们假设我们有一个名叫`--name redis-server`的redis容器。这些文件会定义一个新服务，这个服务会在docker daemon启动后启动。
```bash
[Unit]
Description=Redis container
Requires=docker.service
After=docker.service

[Service]
Restart=always
ExecStart=/usr/bin/docker start -a redis_server
ExecStop=/usr/bin/docker stop -t 2 redis_server

[Install]
WantedBy=default.target
```
如果你打算将其作为系统服务，可以将上诉内容放在`/etc/systemd/system/docker-redis_service.service`文件中
如果你还打算为redis容器传递一些选项[如 `--env`]，那么你需要使用`docker run`,而不是`docker start`,这将会在每次启动服务时创建一个新的容器，而在服务停止时则会停止和删除掉容器。确保您不是使用的`-d`或`分离模式`，因为从`ExecStart`运行的命令需要运行在前台。
```bash
[Service]
...
ExecStart=/usr/bin/docker run --env foo=bar --name redis_server redis
ExecStop=/usr/bin/docker stop -t 2 redis_server
ExecStopPost=/usr/bin/docker rm -f redis_server
...
```
应用该服务
```bash
systemctl daemon-reload
systemctl start docker-redis_server.service
```

允许上面的服务开机启动
```bash
systemctl enable docker-redis_server.service
```
