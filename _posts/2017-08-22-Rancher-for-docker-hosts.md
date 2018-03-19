---
layout: post
title: High Availability Ranchers for docker hosts
categories: [rancher,docker]
description: 记录搭建rancher平台
keywords: rancher,docker
---

### 前言

> 由于公司目前有部分业务跑在docker上，但又还没有上诸如:swarm,kubernetes之类的容器编排工具，但是又想要能对docker主机及容器进行一个简单可视化的管理，筛选来，筛选去，发现无论是:DockerUI、Shipyard、portainer还是Daocloud都不能符合我们的心意，最终决定使用`rancher`来进行管理。

<!--more-->

### Rancher是什么

借用官方文档里面的话来说，Rancher是一个开源的容器管理平台。

实际上Rancher能够整合目前市面上大多数的容器编排工具，如：swarm、kubernetes、mesos等，而且，它本身最拥有一个最基础的编排工具cattle，本次我们使用的就是它的cattle编排器。

### 环境准备

所有系统均为`CentOS 7.2`

| IP | ROLE |
| :--- | :--- |
| 172.30.33.44 | Rancher Server 01 |
| 172.30.33.45 | Rancher Server 02 |
| 172.30.33.227 | External MySQL Server |
| 172.30.33.183 | Nginx LB |

所需镜像如下

| IMAGE | VERSION |
| :--- | :--- |
| rancher/server | latest |
| rancher/agent | v1.2.5 |
| rancher/scheduler | v0.8.2 |
| rancher/healthcheck | v0.3.1 |
| rancher/dns | v0.15.1 |
| rancher/metadata | v0.9.3 |
| rancher/net | v0.11.7 |
| rancher/net | holder |
| rancher/network-manager | v0.7.7 |

### Rancher Server

##### Ecternal mySQL Server

使用如下命令，创建`cattle`数据库和`cattle`用户

```bash
CREATE DATABASE IF NOT EXISTS cattle COLLATE = 'utf8_general_ci' CHARACTER SET = 'utf8';
GRANT ALL ON cattle.* TO 'cattle'@'%' IDENTIFIED BY 'cattle';
GRANT ALL ON cattle.* TO 'cattle'@'localhost' IDENTIFIED BY 'cattle';
```

##### Rancher Server

在每台Node上执行如下命令

```bash
docker run -d --restart=unless-stopped -p 8080:8080 -p 9345:9345 rancher/server \
     --db-host myhost.example.com --db-port 3306 --db-user username --db-pass password --db-name cattle \
     --advertise-address <IP_of_the_Node>
```

> **注意：** 如果你改变了`-p 8080:8080`端口，则需要额外添加一个参数`--addvertise-http-port <host_port>`

##### Nginx LB

在nginx的Vhost中添加如下两个文件

rancher-upstream.conf

```bash
upstream rancher-server {
        server 172.30.33.44:8080;
        server 172.30.33.45:8080;
}
map $http_upgrade $connection_upgrade {
    default Upgrade;
    ''      close;
}         
```

rancher.conf

```bash
server {
        listen 80;
        server_name rancher.quark.com;

        access_log  logs/rancher_access.log main;
        location / {
            #internal;
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header X-Forwarded-Port $server_port;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_pass http://rancher-server;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection $connection_upgrade;
        proxy_read_timeout 900s;
        }
    }       
```

### Rancher agent

上面的步骤完成后，我们来访问我们的rancher ui，然后来添加host
![](/images/posts/rancher-add-agent.png)

依次添加完你的docker host之后，如下图
![](/images/posts/rancher-ui.png)

### 后记

rancher上还有很多功能，感兴趣的同学可以去自行研究下，我这里只是一个临时的需求，所以就不做过多的实例了。
