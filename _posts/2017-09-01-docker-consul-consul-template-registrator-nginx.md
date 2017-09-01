---
layout: post
title: docker+consul+consul-template+registrator+nginx 的容器服务注册发现
categories: [consul,docker,registrator]
description: 基于consul的容器服务注册发现
keywords: consul,docker,registrator
---

### 前言

> 以前了解过一段时间的consul，只知道consul是一个服务发现的工具，但是具体是怎么注册的，又是怎么服务发现的，一点也不清楚，这次趁着研究kubernetes的服务发现，顺带研究了下consul，在此记录下来。

### 概念

简单来说，consul是一个提供服务注册、服务发现、键值存储、健康检查的工具，并且它支持多数据中心。

举个简单的例子，假若我们有一个暴露REST API的服务，为了高可用，我们决定为该服务提供3个服务实例，但是每个容器的地址和端口都是随机的，那么我们的服务之间怎么通信呢，我们又该怎么在前端LB上添加我们的后端服务呢？这时候就需要用到我们的服务发现工具consul了，其实还有很多其他的服务发现工具，比如etcd，zookeeper等等，这里我们重点说下consul。

* 服务之间的通信，这个很简单，我们通过LB即可，所有的服务之间的互相访问，都通过LB即可，我们只需要确定每个服务对于的域名即可。
* 在LB上动态添加后端，这个通过consul-template+consul+registrator即可,consul-template会监控consul中的对应内容，然后根据consul模板文件生成新的配置

该实验所需的所有配置文件内容都放在我的[Github](https://github.com/chinakevinguo/docker-consul.git)上:

> 该项目提供了一种简单的方法，使用consul-template将consul中的值生成具体所需的配置文件，并且实时监控consul，根据模板文件生成最新的配置文件，然后运行某些指定命令。

由于环境有限，我下面所有的实验都是在一台机器上用docker完成，你只需要修改对应的IP地址即可

### Consul

提供服务注册和服务发现

```bash
# 注意这里我指定了-client是为了方便我通过外网访问consul ui

docker run -d --net=host --name consul -h consul -e 'CONSUL_LOCAL_CONFIG={"skip_leave_on_interrupt": true}' consul agent -server -client $HOST_IP -ui -bootstrap
```

### Registrator

将宿主机上的容器自动注册到consul中

```bash

docker run -d --net=host -v /var/run/docker.sock:/tmp/docker.sock --name registrator -h registrator gliderlabs/registrator:latest -internal consul://$HOST_IP:8500
```

### Nginx with consul-template

利用consul-template监控consul，根据模板生成新的配置，并提供负载均衡

将上面地址中的内容clone到本地,build nginx-consul镜像

```bash
git clone https://github.com/chinakevinguo/docker-consul.git
cd docker-consul
chmod +x start.sh
docker build -t docker-consul .
```

启动nginx

```bash
docker run -p 8080:80 -d --name nginx -e CONSUL_URL=$HOST_IP:8500 --volume ~/docker-consul/service.ctmpl:/templates/service.ctmpl  nginx-consul
```

### 启动一些服务实例

具体提供服务的实例

```bash
docker run -d -P --name node1 -h node1 jlordiales/python-micro-service:latest
docker run -d -P --name node2 -h node2 jlordiales/python-micro-service:latest
docker run -d -P --name node3 -h node3 jlordiales/python-micro-service:latest
```

启动完成后，我们来看看nginx中是否已经动态添加了这些后端呢

```bash
$ docker exec -ti nginx cat /etc/nginx/conf.d/service.conf
upstream python-service {
  least_conn;
  server 172.17.0.5:5000 max_fails=3 fail_timeout=60 weight=1;
  server 172.17.0.6:5000 max_fails=3 fail_timeout=60 weight=1;
  server 172.17.0.7:5000 max_fails=3 fail_timeout=60 weight=1;
  server 172.17.0.8:5000 max_fails=3 fail_timeout=60 weight=1;

}

server {
  listen 80 default_server;

  charset utf-8;

  location /{
    proxy_pass http://python-service;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
  }
}

```

然后我们来看看我们的consul ui中展现了那些内容呢

![](/images/posts/consul-ui.png)

我们发现我们的python-micro-service 服务目前有4个，而且当我们通过如下命令访问的时候，也是4个轮询着被访问

```bash
$ while true; do curl $HOST_IP:8080; echo ----; sleep 1; done;
Hello World from node4----
Hello World from node1----
Hello World from node2----
Hello World from node3----
```

这时候，如果我马上停掉其中一个呢,我们发现consul ui中的服务也相应的减少，而且用命令访问的时候，也已经变成了3个

![](/images/posts/consul-ui-service.png)

```bash
$ while true; do curl $HOST_IP:8080; echo ----; sleep 1; done;
Hello World from node1----
Hello World from node2----
Hello World from node3----
```

### Conclusion

文章写的比较简单，其实就是通过一个简单的实验，了解了下consul的服务注册，服务发现，以及如何使用consul-template来动态的生成对应的配置文件，而关于服务注册，我们使用的是registrator，也许你的项目需要调用consul的HTTP API来注册也说不定，具体你可以去[consul官网](https://www.consul.io/)了解更多。
