---
layout: post
title: Docker基础-bridge绑定容器端口
categories: [docker]
description: 在bridge上绑定容器端口
keywords: docker
---
本章节主要讲解了在Docker默认的`bridge`网络上绑定容器端口，这个`bridge`网络是在docker安装的时候，系统默认创建的。
> **Note:** docker允许创建除了默认`bridge`网络之外的bridge网络

<!--more-->

默认情况下docker容器可以连接到外网，但是外网不能连接到容器。由于docker服务器在启动的时候会创建主机上的iptables规则，所以每个容器都会有自己的nat规则
```bash
$ iptables -t nat -L -n
Chain POSTROUTING (policy ACCEPT)
target     prot opt source               destination         
MASQUERADE  all  --  172.17.0.0/16        0.0.0.0/0           
MASQUERADE  tcp  --  172.17.0.3           172.17.0.3           tcp dpt:5000
MASQUERADE  tcp  --  172.17.0.4           172.17.0.4           tcp dpt:5000
```
docker服务器会创建一个伪装规则，使容器连接到外部IP地址。

如果希望容器能接入连接，你需要在调用`docker run `的时候指定特别的选项
一.你可以在运行`docker run `的时候指定`-P`或`--publish-all=true|false`来将Dockerfile中EXPOSE暴露的端口映射到主机上的随机端口
二.或者使用`--expose <port>`来额外指定暴露出来的端口
三.映射还可以使用`-p`或`--publish=`来精确指定暴露容器的某个端口到宿主机上某个IP的的某个端口
我们可以使用`docker port`来查看端口的信息，而这个宿主机上随机端口是配置在`/proc/sys/net/ipv4/ip_local_port_range`，范围大概是32768-61000.



无论哪种方式，你都可以通过检查`NAT`表来确认
```bash
$ iptables -t nat -L -n
# 这是允许容器访问外网
Chain POSTROUTING (policy ACCEPT)
target     prot opt source               destination         
MASQUERADE  all  --  172.17.0.0/16        0.0.0.0/0           
MASQUERADE  tcp  --  172.17.0.3           172.17.0.3           tcp dpt:5000
MASQUERADE  tcp  --  172.17.0.4           172.17.0.4           tcp dpt:5000

# 这是允许外部访问容器
Chain DOCKER (2 references)
target     prot opt source               destination         
RETURN     all  --  0.0.0.0/0            0.0.0.0/0           
DNAT       tcp  --  0.0.0.0/0            0.0.0.0/0            tcp dpt:32768 to:172.17.0.3:5000
DNAT       tcp  --  0.0.0.0/0            0.0.0.0/0            tcp dpt:32769 to:172.17.0.4:5000
DNAT       tcp  --  0.0.0.0/0            192.168.1.1          tcp dpt:9292 to:172.17.0.11:80
```

我们可以看到docker在`0.0.0.0`或`192.168.1.1`接口上公开了指定的端口或随机端口，对应映射到容器内的端口

如果你想要docker端口转发绑定到一个特定的IP地址，你可以编辑docker启动配置文件，添加`--ip`选项，当然也可以直接使用`-p`选项指定
```bash
$ docker ps
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS                      NAMES
a894fe52388a        test                "/bin/bash"         3 seconds ago       Up 1 seconds        192.168.1.1:9298->80/tcp   romantic_shannon
```
