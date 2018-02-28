---
layout: post
title: Docker基础-当docker daemon停止时依然保持容器运行
categories: [docker]
description: docker 当docker daemon停止时依然保持容器运行
keywords: docker
---
默认情况下，当docker daemon停止，它会关闭所有正在运行的容器。从docker1.12起，你可以配置当docker daemon不可用时依然保持容器继续运行。`live restore`选项有助于减少由于docker daemon崩溃，中断或升级而导致的容器停机时间。

> **Note:** live restore不支持windows容器，但是它支持运行在windows上的linux容器。
<!--more-->

# 开启live restore选项

有两种方式开启`live restore`:
* 如果docker daemon正在运行并且你不想停止它，你可以添加配置到docker daemon的配置文件。例如：在linux系统上默认的配置文件是/etc/docker/daemon.json
```bash
{
"live-restore": true
}
```
你必须传递一个`SIGHUP`信号给daemon进程来重载配置。更多有关使用config.json来配置docker daemon的信息，可以参考[daemon configuration file](https://docs.docker.com/engine/reference/commandline/dockerd/#daemon-configuration-file)
* 在使用dockerd启动时指定`--live-restore`选项
```bash
$ sudo dockerd --live-restore
```
# 升级docker daemon 时的Live restore
live restore支持当升级docker daemon后，还原容器到docker

# 重启时的live restore
live restore只支持还原到和原docker daemon一样的docker daemon。比如：如果新的docker daemon使用不同的网桥IP重新启动，则live restore不起作用。

# live restore 对正在运行的容器的影响
长时间缺少docker daemon可能会影响正在运行的容器。容器写入FIFO日志在daemon消耗时。如果daemon不能用于输出，缓冲区将填满并阻止对日志的进一步写入，因为阻塞该进程直到有更多的可用空间，默认缓冲区大小通常为64K。
你必须重启docker来刷新buffers
你可以修改`/proc/sys/fs/pipe-max-size`来修改内核buffer 大小

# live restore 和 swarm 模式
live restore 和docker swarm模式不兼容。当docker 运行在swarm模式下时，是由编排功能来管理任务并使容器根据服务规范运行的。
