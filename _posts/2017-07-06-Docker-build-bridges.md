---
layout: post
title: Docker基础-自定义docker bridge
categories: [docker]
description: 自定义docker bridge
keywords: docker
---
这篇主要解释了怎么绑定你自己的bridge来替换docker默认的bridge，这个默认的bridge网络是在安装docker的时候自动安装的。

> **Note:** docker允许你创建默认bridge网络之外的自定义bridge网络。
<!--more-->
你可以在启动docker之前设定你自己的bridge网络，并使用`-b BRIDGE` or `--bridge=BRIDGE` 来告诉docker使用它。如果已经有偶docker启动并运行，并且使用的是docker0的默认配置，你可以直接创建自定义的bridge网络，并重启docker，或者停止服务之后删除接口
```bash
# 停止docker服务，并删除docker0接口
$ sudo service docker stop
$ sudo ip link set dev docker0 down
$ sudo brctl delbr docker0
$ sudo iptables -t nat -F POSTROUTING
```

然后,在创建docker服务之前,创建自己的网桥,并给她任何你想要的配置.这里,我们创建了一个足够简单的bridge,我们可以使用上一节中的选项来自定义docker0,这就足够了.
```bash
# create our own bridge
$ brctl addbr bridge0
$ ip addr add 192.168.5.1/24 dev bridge0
$ ip link set dev bridge0 up

# 配置bridge up并且running

$ ip addr show bridge0

4: bridge0: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state UP group default
    link/ether 66:38:d0:0d:76:18 brd ff:ff:ff:ff:ff:ff
    inet 192.168.5.1/24 scope global bridge0
       valid_lft forever preferred_lft forever

# 配置docker启动的配置文件,修改成如下类容
$ vi /usr/lib/systemd/system/docker.service
ExecStart=/usr/bin/dockerd -b=bridge0

# 重启docker
systemctl start docker

# 看看新的NAT是否启用
$ iptables -t nat -L -n
Chain POSTROUTING (policy ACCEPT)
target     prot opt source               destination         
MASQUERADE  all  --  192.168.5.0/24       0.0.0.0/0  
```
我们看到docker服务已经成功重启，现在可以将新的容器绑定到新的bridge，然后我们新建一个容器看看
```bash
$ docker run -tid test
$ docker inspect clever_spence
...
"Gateway": "192.168.5.1",
"IPAddress": "192.168.5.2",
...
```
我们发现，我们刚创建的bridge已经生效。
我们可以使用`brctl`来对bridge进行查看，删除，添加等操作。
```bash
$ brctl show
bridge name	bridge id		STP enabled	interfaces
bridge0		8000.f600adbf94d5	no		veth79dcc4b
```
