---
layout: post
title: docker中的一些组件
categories: [docker, runC, containerd]
description: docker中的一些组件
keywords: docker,runC,containerd,containerd-shim
---

### docker主要组件

安装docker，实际上就是安装了docker客户端、dockerd等一系列组件，其中比较重要的有下面几个

![](/images/posts/containerd.png)

#### docker CLI(docker)

docker 程序是一个客户端工具，用来把用户的请求发送给docker daemon(dockerd)

#### dockerd

docker daemon(dockerd)，docker服务端，一般也会被称为docker engine

#### docker-containerd

带API的容器运行时

containerd主要职责是镜像管理（镜像、元信息等）、容器执行（调用最终运行时组件执行）。

containerd向上为Docker Daemon提供了gRPC接口，使得Docker Daemon屏蔽下面的结构变化，确保原有接口向下兼容。

containerd向下通过containerd-shim结合runC，使得引擎可以独立升级，避免之前Docker Daemon升级会导致所有容器不可用的问题

当docker daemon启动后，dockerd和docker-containerd进程就一直存在。

```bash
[root@kevinguo my_container]# ps -ef |grep docker
root      5261     1  0 11:48 ?        00:00:03 /usr/bin/dockerd
root      5266  5261  0 11:48 ?        00:00:10 docker-containerd --config /var/run/docker/containerd/containerd.toml
```

当启动容器之后，docker-containerd会创建子进程docker-containerd-shim进程

```bash
root      5261     1  0 11:48 ?        00:00:03 /usr/bin/dockerd
root      5266  5261  0 11:48 ?        00:00:10 docker-containerd --config /var/run/docker/containerd/containerd.toml
root      5484  5266  0 11:48 ?        00:00:00 docker-containerd-shim -namespace moby -workdir /var/lib/docker/containerd/daemon/io.containerd.runtime.v1.linux/moby/eaad9500e9a03473933179362983dd4615ba075062b184b0c4e8a3a34abacf34 -address /var/run/docker/containerd/docker-containerd.sock -containerd-binary /usr/bin/docker-containerd -runtime-root /var/run/docker/runtime-runc
```

##### 安装containerd

containerd会调用runc，所以我们在安装containerd之前，需先安装runc

从[官网](https://github.com/containerd/containerd/releases/download/v1.1.0/containerd-1.1.0.linux-amd64.tar.gz)下载最新的稳定版本

然后把下载的包解压到`/usr/local`下

```bash
sudo tar -C /usr/local -xf containerd-1.1.0.linux-amd64.tar.gz
```

配置containerd作为系统服务

```bash
mkdir /etc/containerd
containerd config default > /etc/containerd/config.toml

vi /usr/lib/systemd/system/containerd.service

[Unit]
Description=containerd container runtime
Documentation=https://containerd.io
After=network.target

[Service]
ExecStartPre=/sbin/modprobe overlay
ExecStart=/usr/local/bin/containerd
Delegate=yes
KillMode=process
LimitNOFILE=1048576
# Having non-zero Limit*s causes performance problems due to accounting overhead
# in the kernel. We recommend using cgroups to do container-local accounting.
LimitNPROC=infinity
LimitCORE=infinity

[Install]
WantedBy=multi-user.target
```

启动服务，然后尝试着用ctr命令来启动容器

```bash
systemctl daemon-reload containerd
systemctl start containerd

# containerd的默认命令ctr

ctr images pull docker.io/library/busybox:latest
ctr run -t docker.io/library/busybox:latest mybusybox

# 列出容器
[root@kevinguo ~]# ctr c ls
CONTAINER    IMAGE                               RUNTIME
mybusybox    docker.io/library/busybox:latest    io.containerd.runtime.v1.linux

# 列出任务
[root@kevinguo ~]# ctr t ls
TASK         PID      STATUS
mybusybox    16359    RUNNING
```

删除容器之前先要kill掉容器进程，然后再用`ctr c rm mybusybox`的命令来删除

#### docker-containerd-shim

它是containerd的一个组件，是容器的运行时载体，我们在宿主机上看到的shim，正是代表着一个个通过containerd启动的docker容器，它允许容器运行时在容器启动之后退出，即便在containerd和dockerd都挂掉的情况下，容器的标准io和其他描述文件都是可用的

#### runC

容器运行时

RunC 是一个轻量级的工具，它是用来运行容器的，只用来做这一件事，并且这一件事要做好。我们可以认为它就是个命令行小工具，可以不用通过 docker 引擎，直接运行容器。事实上，runC 是标准化的产物，它根据 OCI 标准来创建和运行容器。而 OCI(Open Container Initiative)组织，旨在围绕容器格式和运行时制定一个开放的工业化标准。

我们可以在运行dockerd的时候使用`--add-runtime`来指定其他的运行时

在容器启动的过程中，docker-runc 进程是作为 docker-containerd-shim 的子进程存在的。docker-runc 进程根据配置找到容器的 rootfs 并创建子进程 bash 作为容器中的第一个进程。当这一切都完成后 docker-runc 进程退出，然后容器进程 bash 由 docker-runc 的父进程 docker-containerd-shim 接管

##### 安装runc

runc是golan的项目，因此在编译它之前，咱们先安装一下golan，这里我是在centos上，直接用命令安装

```bash
yum install golan -y
```

[参考官方文档](https://github.com/opencontainers/runc)

安装必要的依赖包

```bash
yum install libseccomp libseccomp-devel -y
```

在你的golan的src目录下，创建github.com/opencontainers，如果你也是用yum 安装的golan，那么目录如下

```bash
mkdir -p /usr/lib/golang/src/github.com/opencontainers
```

获取runc的代码，并且编译安装

```bash
git clone https://github.com/opencontainers/runc
cd runc
make
make install

[root@kevinguo src]# runc --version
runc version 1.0.0-rc5+dev
commit: 2c632d1a2de0192c3f18a2542ccb6f30a8719b1f
spec: 1.0.0
```

##### 使用runC/docker-runc来运行busybox容器

先准备一个工作目录，所有的操作都是在该目录下执行

```bash
mkdir mycontainer
```

接着，准备容器镜像的文件系统，我们从docker镜像中获取

```bash
cd mycontainer
mkdir rootfs
docker export $(docker create busybox) |tar -C rootfs -xvf -
```

有了rootfs之后，我们还需要一个符合OCI标准的配置文件config.json，我们可以使用docker-runc来生成

```bash
docker-runc spec
```

接着，我们就可以使用runC来操作了

```bash
docker-runc run busybox

# 切换到另一个终端来查看
[root@kevinguo ~]# docker-runc list
ID          PID         STATUS      BUNDLE              CREATED                          OWNER
busybox     8754        running     /root/mycontainer   2018-07-04T06:39:50.137725285Z   root

[root@kevinguo ~]# docker-runc ps busybox
UID        PID  PPID  C STIME TTY          TIME CMD
root      8754  8745  0 15:39 pts/0    00:00:00 sh
```


我们还可以尝试着修改config.json，让容器后台运行

```json
"terminal": false,
"args": [
        "sleep","20"
],
```

```bash
[root@kevinguo mycontainer]# docker-runc create mybusybox
[root@kevinguo mycontainer]# docker-runc start mybusybox
[root@kevinguo mycontainer]# docker-runc list
ID          PID         STATUS      BUNDLE              CREATED                          OWNER
busybox     8754        running     /root/mycontainer   2018-07-04T06:39:50.137725285Z   root
mybusybox   8886        running     /root/mycontainer   2018-07-04T06:42:14.404232848Z   root
```





> 就现状而言，虽然containerd1.1已经可以直接被kubernetes使用了，但是相对而言，我们还是建议先使用docker这个容器运行时，所以不论是cri还是cri-o，简单了解下就好
