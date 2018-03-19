---
layout: post
title: Docker基础-docker在各个平台的基本配置
categories: [docker]
description: docker在各个平台的基本配置
keywords: docker
---

在安装完docker之后，docker daemon会用默认的配置来运行。

在生产环境中，系统管理员通常会根据需求来配置docker，在大多数例子中，系统管理员配置会配置进程管理器，如：`sysvinit`,`upstart`或`systemd`来管理docker的启动和停止。
<!--more-->
# 直接运行docker daemon

我们可以直接用`dockerd`命令来直接运行docker daemon.默认是监听在`unix socket`
`unix:///var/run/docker.sock`
```bash
$ dockerd

INFO[0000] +job init_networkdriver()
INFO[0000] +job serveapi(unix:///var/run/docker.sock)
INFO[0000] Listening for HTTP on unix (/var/run/docker.sock)
...
...
```

# 直接配置docker daemon

如果你是直接运行`dockerd`命令，而非使用进程管理器(systemctl start docker)，你可以直接将配置选项附加到`docker run`命令。其他配置选项可传递给docker daemon来配置
配置选项如下：

| Flag | Description |
| :---- | :--------- |
| -D,--debug=false | 开启或关闭debug模式，默认是关闭的 |
| -H,--host=[] | Daemon socket[s] 连接到哪 |
| --tls=false | 开启或关闭TLS，默认是关闭的 |
这里有一个使用配置选项运行docker daemon的例子：
```bash
$ dockerd -D \
--tls=true \
--tlscert=/var/docker/server.pem \
--tlskey=/var/docker/serverkey.pem \
-H tcp://192.168.59.3:2376
```
这些选项是：
* 开启debug模式 `-D`
* 开启`tls`并指定证书 `--tlscert`和`--tlskey`
* 监听连接`tcp://192.168.59.3:2376`

命令行可参考[complete list of daemon flags](https://docs.docker.com/engine/reference/commandline/dockerd/)

# Daemon debugging

如上所诉，开启debug模式来允许管理员或操作者来获取docker daemon运行时的信息。如果面对一个没有响应的daemon，管理员可以通过向Docker daemon发送`SIGUSR1`信号来强制所有线程的完整堆栈跟踪添加到daemon的日志中。在linux上通常使用`kill`命令。例：`kill -USR1 <daemon-pid>`发送`SIGUSR1`到daemon，这会导致堆栈被添加到daemon日志中。

> **Note:** 日志级别至少是`info`及以上，默认日志界别是`info`

在处理`SIGUSR1`信号并将堆栈跟踪转存到日志和欧，daemon将继续运行，堆栈跟踪可用于确定daemon所有线程和goroutines的状态

# 在centos上配置docker

在CENTOS 6.X和RHEL 6.X中，我们在`/etc/sysconfig/docker`文件中配置docker daemon，我们可以通过指定`other_args`变量来实现。短时间内，在Centos7.x 和RHEL 7.x我们使用`OPTIONS`变量值，不再推荐直接使用`systemd`。

1.登录到你的系统，使用root账户，或者使用sudo
2.创建`/etc/systemd/system/docker.service.d`目录
```bash
$ sudo mkdir /etc/systemd/system/docker.service.d
```
3.创建`/etc/systemd/system/docker.service.d/docker.conf`文件
```bash
$ sudo mkdir /etc/systemd/system/docker.service.d/docker.conf
```
4.打开这个docker.conf文件
```bash
$ sudo vi /etc/systemd/system/docker.service.d/docker.conf
```
5.覆盖从`docker.service`文件复制过来的`ExecStart`，用以自定义`docker daemon`。要修改`ExecStart`配置，必须先指定一个空配置，然后再知道你该一个新配置，如下所示：
```bash
[Service]
ExecStart=
ExecStart=/usr/bin/dockerd -H fd:// -D --tls=true --tlscert=/var/docker/server.pem --tlskey=/var/docker/serverkey.pem -H tcp://192.168.59.3:2376
```
6.保存关闭文件
7.重新载入改变
```bash
$ sudo systemctl daemon-reload
```
8.重启docker daemon
```bash
$ sudo systemctl restart docker
```
9.检查docker daemon是否运行
```bash
$ ps aux | grep docker | grep -v grep
```

# logs
docker 在centos7.x上是将日志保存在`/var/log/messages` 中的，我们也可以使用`journalctl -u docker`来查看docker的日志
```bash
[root@node01 log]# journalctl -u docker
-- Logs begin at Tue 2017-01-17 01:11:58 UTC, end at Tue 2017-01-17 02:52:15 UTC. --
Jan 17 01:22:56 node01 systemd[1]: Starting Docker Application Container Engine...
Jan 17 01:22:57 node01 dockerd[2141]: time="2017-01-17T01:22:57.477500482Z" level=info msg="libcontainerd: new containerd process, pid: 2145"
Jan 17 01:22:59 node01 dockerd[2141]: time="2017-01-17T01:22:59.035780325Z" level=warning msg="devmapper: Usage of loopback devices is strongly
Jan 17 01:22:59 node01 dockerd[2141]: time="2017-01-17T01:22:59.071446777Z" level=warning msg="devmapper: Base device already exists and has fi
Jan 17 01:22:59 node01 dockerd[2141]: time="2017-01-17T01:22:59.077953805Z" level=info msg="[graphdriver] using prior storage driver \"devicema
Jan 17 01:22:59 node01 dockerd[2141]: time="2017-01-17T01:22:59.442654952Z" level=info msg="Graph migration to content-addressability took 0.00
Jan 17 01:22:59 node01 dockerd[2141]: time="2017-01-17T01:22:59.443436730Z" level=warning msg="mountpoint for pids not found"
Jan 17 01:22:59 node01 dockerd[2141]: time="2017-01-17T01:22:59.443929680Z" level=info msg="Loading containers: start."

```
