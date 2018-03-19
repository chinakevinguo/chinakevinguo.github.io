---
layout: post
title: Docker基础-docker在自定义的网络上定义DNS
categories: [docker]
description: 自定义dns
keywords: docker
---
# 在自定义网络上的内嵌dns
这一章主要讲了在用户自定义网络中，容器内嵌dns的操作。连接到用户自定义网络的容器的DNS查询和连接到默认网桥网络的容器的查询方式不同。
> **Note:** 为了保持向后的兼容性，默认网桥中的DNS配置保留，没有进行更改。有关默认网桥中DNS配置的详细信息，可以参考[DNS in default bridge network](https://docs.docker.com/engine/userguide/networking/default_network/configure-dns/)

<!--more-->
截至到Docker1.10，Docker守护进程实现了一个内嵌的DNS服务器，该服务器为那些提供了有效名称、网络别名或连接别名的容器，提供内置服务发现。Docker管理容器内的DNS配置会从一个版本变成下一个版本，所以，你不应该在容器里管理如`/etc/hosts`,`/etc/resolv.conf`之类的文件，而应该将它们单独进行管理。
影响容器域名服务的各种选项如下：

| Options | Description |
| :------ | :---------  |
| --name=CONTAINER-NAME | 使用`--name`配置的容器名，用于在用户自定义的网络中发现容器，内嵌的DNS服务器维护容器的名称和IP地址之间的隐射 |
| --network-alias=ALIAS | 除了上述的`--name`之外，容器在用户自定义网络中还可以定义一个或多个网络别名。内嵌的DNS服务器畏惧所有容器别名和其在特定自定义网络上的IP地址之间的隐射。一个容器可以在不同的网络上拥有不同的别名 |
| --link=CONTAINER_NAME:ALIAS | 在运行容器的时候使用`--link`为内嵌DNS提供一个ALIAS条目，该条目指向由CONTAINER_NAME标识的容器的IP地址 |
| --dns=[IP_ADDRESS...] | 如果内嵌DNS无法解析容器的请求，那么会通过--dns指定的IP地址进行转发DNS查询，这些--dns ip地址由内嵌DNS服务器管理，不会在容器的/etc/resolv.conf文件中更新 |
| --dns-search=DOMAIN | 指定dns的搜索域，当容器设置了搜索域时，DNS不仅查找主机，还查找域，当然这个设置，也不会在容器的/etc/resolv.conf文件中更新 |
| --dns-opt=OPTION... | 设置DNS解析器的使用选项，由内嵌的DNS服务器管理，不会在容器中更新 |

在没有指定上面的那些选项时，docker使用宿主机的/etc/resolv.conf，在执行操作时，会将所有localhost的域名解析全部过滤掉。
过滤掉localhost域名解析是有必要的，因为主机上的所有localhost地址都无法从容器网络访问，过滤掉后，如果容器的/etc/reslov.conf文件中没有更多nameservers，docker daemon会添加GOOGLE的dns到容器中

> **Note:** 如果你非要容器能访问localhost的域名解析，那你就必须在主机上修改你的dns服务，以侦听可从容器内访问的非本地主机地址
