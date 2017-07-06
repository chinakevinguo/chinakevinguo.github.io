---
layout: post
title: Docker基础-自定义docker daemon
categories: [docker]
description: 自定义docker daemon
keywords: docker
---

很多linux发行版使用systemd来启动docker daemon。这篇文档主要介绍了几个例子来展示如何自定义docker 的设置。

# 启动docker daemon
安装完docker后，你需要启动docker daemon
```bash
$ sudo systemctl start docker
# 在老版本上
$ sudo service docker start
```
如果你想开机启动docker
```bash
$ sudo systemctl enable docker
# 老版本
$ sudo chkconfig docker on
```
<!--more-->
# 自定义docker daemon 选项
有很多种方法为你的docker daemon配置daemon标志和环境变量。
推荐的方法是使用systemd的插件文件[如[systemd.unit](https://www.freedesktop.org/software/systemd/man/systemd.unit.html)所描述的]。这些插件文件是一些在本地`/etc/systemd/system/docker.service.d`目录下的名为`<something>.conf`文件。
但是，如果你之前使用过一个包含`EnvironmentFile`[通常是`/etc/sysconfig/docker`]的包，那么为了向后兼容，你可将具有`.conf`扩展名的文件放到`/etc/systemd/system/docker.service.d`目录中，包含如下内容：
```bash
[Service]
EnvironmentFile=-/etc/sysconfig/docker
EnvironmentFile=-/etc/sysconfig/docker-storage
EnvironmentFile=-/etc/sysconfig/docker-network
ExecStart=
ExecStart=/usr/bin/dockerd $OPTIONS \
          $DOCKER_STORAGE_OPTIONS \
          $DOCKER_NETWORK_OPTIONS \
          $BLOCK_REGISTRY \
          $INSECURE_REGISTRY
```

检查docker.service
```bash
$ systemctl show docker | grep EnvironmentFile

EnvironmentFile=-/etc/sysconfig/docker (ignore_errors=yes)
```

找出service 文件的位置
```bash
$ systemctl show --property=FragmentPath docker

FragmentPath=/usr/lib/systemd/system/docker.service

$ grep EnvironmentFile /usr/lib/systemd/system/docker.service

EnvironmentFile=-/etc/sysconfig/docker
```
`/usr/lib/systemd/system `或`/lib/systemd/system`包含默认选项，建议不要修改。

# 运行时的目录和存储驱动

你可能希望通过将docker image，容器和volumes移动到单独的分区来控制用于docker image，容器和volumes的磁盘空间。
在这个例子中，我们假设你的`docker.service`文件像这样：
```bash
[Unit]
Description=Docker Application Container Engine
Documentation=https://docs.docker.com
After=network.target

[Service]
Type=notify
# the default is not to use systemd for cgroups because the delegate issues still
# exists and systemd currently does not support the cgroup feature set required
# for containers run by docker
ExecStart=/usr/bin/dockerd
ExecReload=/bin/kill -s HUP $MAINPID
# Having non-zero Limit*s causes performance problems due to accounting overhead
# in the kernel. We recommend using cgroups to do container-local accounting.
LimitNOFILE=infinity
LimitNPROC=infinity
LimitCORE=infinity
# Uncomment TasksMax if your systemd version supports it.
# Only systemd 226 and above support this version.
#TasksMax=infinity
TimeoutStartSec=0
# set delegate yes so that systemd does not reset the cgroups of docker containers
Delegate=yes
# kill only the docker process, not all processes in the cgroup
KillMode=process

[Install]
WantedBy=multi-user.target
```
这将允许我们通过放置文件(如上所述)，通过在`/etc/systemd/system/docker.service.d`目录中放置一个包含以下内容的文件来添加额外的标志：
```bash
[Service]
ExecStart=
ExecStart=/usr/bin/dockerd --graph="/mnt/docker-data" --storage-driver=overlay
```
你还可以在这个文件中设置其他环境变量，比如:`HTTP_PROXY`环境变量

修改`ExecStart`配置，先指定一个空配置，然后指定一个新的配置
```bash
[Service]
ExecStart=
ExecStart=/usr/bin/dockerd --bip=172.17.42.1/16
```

如果你没有指定空配置，那么可能会报错：
```bash
docker.service has more than one ExecStart= setting, which is only allowed for Type=oneshot services. Refusing.
```

# HTTP proxy
这个例子会覆盖默认的`docker.service`文件
如果你在代理服务器后面，那么你需要将代理配置添加到docker systemd service文件中
1.为docker服务创建一个systemd插件目录：
```bash
$ mkdir -p /etc/systemd/system/docker.service.d
```

2.创建一个`/etc/systemd/system/docker.service.d/http-proxy.conf`配置文件,并添加`HTTP_PROXY`环境变量
```bash
[Service]
Environment="HTTP_PROXY=http://proxy.example.com:80/"
```

3.如果内部有不需要代理访问的docker Registry，那么你可以通过`NO_PROXY`来指定
```bash
Environment="HTTP_PROXY=http://proxy.example.com:80/" "NO_PROXY=localhost,127.0.0.1,docker-registry.somecorporation.com"
```

4.重新载入
```bash
$ sudo systemctl daemon-reload
```

5.检查配置是否加载
```bash
$ sudo systemctl restart docker
```

# 手动创建systemd 单元文件
当安装没有包的二进制文件时，你可能想要将docker和systemd集成。为此，只需要将两个端圆文件(service 和 socket)从[github repository](https://github.com/docker/docker/tree/master/contrib/init/systemd) 安装到/etc/systemd/system即可
