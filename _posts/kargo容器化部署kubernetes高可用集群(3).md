---
layout: post
title: kargo容器化部署kubernetes高可用集群(3)
categories: [kubernetes, docker]
description: kargo容器化部署kubernetes高可用集群
keywords: kargo,kubernetes,docker
---

> 上一篇我们详细的剥析了通过kargo生成的各类服务的配置文件，学会了，如何生成证书，如何配置etcd,calico,kubelet，学会了如何配置一个kubernetes的高可用集群。既然集群已经配好了，那么这一章，我们就来学学如何配置一些常用的插件。

其实这一章好像没什么好讲的，无非就是几个yaml文件，只要你集群搭建成功了，直接去[这里](https://github.com/chinakevinguo/kubernetes-custom/tree/master/kubernetes%20%E6%8F%92%E4%BB%B6)下载yml文件运行即可

目前，我所用到的插件如下：

* ingress
* kubedns
* dashboard
* efk
* heapster
<!--more-->

运行结果如下

Dashboard and heapster
![](/images/post/dashboard-heapster.png)

EFK
![](/images/post/kibana.png)

后期再研究新的东西了再加吧，头疼，下班了
