---
layout: post
title: Docker基础-docker常用命令
categories: [docker]
description: docker常用命令
keywords: docker
---
### Docker Command
#### Docker container management
```bash
#运行容器
docker run [OPTIONS] IMAGE [COMMAND] [ARG...]
  -t 分配一个terminal窗口
  -i 允许和容器进行交互
  -d 后台运行一个容器
  -P 将容器内部的所有exposed的端口映射为宿主机上的随意端口
  -p 将容器内部的端口绑定到宿主机上指定的端口
     后面跟<port> ：表示将容器内部的这个端口隐射到宿主机上的随意端口
     后面跟<port1>:<port2> ：表示将<port2容器内部端口>隐射到<port1宿主机端口>
  --name 为容器命名
  --net 为容器指定网络
  -v 为容器挂载卷组
  --volume-driver 指定容器卷组的驱动

$ docker run -d -P --name web -v /src/webapp:/webapp training/webapp python app.py
$ docker run -d -P --name web training/webapp python app.py
$ docker run -d -p 80:5000 trainning/webapp python app.py
$ docker run ubuntu /bin/echo "Hello world"
$ docker run -ti ubuntu /bin/bash
```
<!--more-->
```bash
#启动容器，可同时启动多个容器
docker start [OPTIONS] CONTAINER [CONTAINER...]

$ docker start 215b04f73370
```

```bash
#停止容器运行，可同时停止多个容器
docker stop [OPTIONS] CONTAINER [CONTAINER...]

$ docker stop 215b04f73370
```

```bash
#重启重启，可同时重启多个容器
docker restart [OPTIONS] CONTAINER [CONTAINER...]

$ docker restart 215b04f73370
```

```bash
#删除容器
docker rm [OPTIONS] CONTAINER [CONTAINER...]
  -f 强制删除一个正在运行的容器(实际上是先停止容器，然后再删除容器)
  -v 删除和容器关联的卷

$ docker rm -f 215b04f73370
```

```bash
#列出容器清单
docker ps [OPTIONS]
  -a 列出所有容器,包括已经停止的容器
  -q 值显示容器ID
  -l 列出最近创建的容器
```

```bash
#连接到一个正在运行的容器
docker attach [OPTIONS] [CONTAINER]
```
```bash
#在一个已经运行的容器里运行命令
docker exec [OPTIONS] CONTAINER COMMAND [ARG...]

$ docker exec -ti db1 /bin/bash
```

#### Docker container state management
```bash
#查看容器内部日志
docker logs [OPTIONS] CONTAINER
  -f 容器日志追踪

$ docker logs grave_roentgen
$ docker logs 20348fdf82e4
```

```bash
#查看容器内部运行的进程
docker top CONTAINER
```

```bash
#获取容器、images、task的配置信息和状态信息
docker inspect [OPTIONS] CONTAINER|IMAGE|TASK [CONTAINER|IMAGE|TASK...]
  -f 按照给定的模板来输出结果

$ docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' romantic_mahavira
```

```bash
#查看容器内公开端口隐射在宿主机上的端口
docker port CONTAINER [PRIVATE_PORT[/PROTO]]

$ docker port 215b04f73370
```

#### Docker images management
```bash
#列出image清单
docker images [OPTIONS] [REPOSITORY[:TAG]]
  -a 显示所有image
  -q 只显示image ID
  --digests 显示摘要信息
```

```bash
#拉取image镜像
docker pull [OPTIONS] IMAGENAME[:TAG|@DIGEST]
  -a 将所有tag image下载下来
  --disable-content-trust=true 取消检查image

$ docker pull chinakevinguo/docker-whale:latest
```

```bash
#查找images
docker search [OPTIONS] image

$ docker search sinatra
```

```bash
#登录到docke服务器
docker login [OPTIONS] [SERVER]
  -u 用户名
  -p 密码

$ docker login
Username: chinakevinguo
Password:
Login Succeeded
```

```bash
#将经过修改的容器构建成新的镜像，一般不推荐这种用法
docker commit [OPTIONS] CONTAINER [REPOSITORY[:TAG]]
  -a 作者
  -m 描述

$ docker commit -m "Added json gem" -a "Kevin Guo" goofy_bohr chinakevinguo/sinatra:v2
```

```bash
#通过dockerfile构建镜像
docker build [OPTIONS] PATH | URL
  -t 指定image的REPOSITORY，可以指定多个REPOSITORY
  -f 指定你的dockerfile的名称路径（默认是$PATH/Dockerfile）

$ docker build -t chinakevinguo/sinatra:v3  ~/sinatra
$ docker build -t chinakevinguo/sinatra:v3 -f ~/sinatra/mydockerfile .
```

```bash
#为image打tag，也可以算是为images重命名
docker tag IMAGE[:TAG] IMAGE[:TAG]

$ docker tag chinakevinguo/sinatrasdfa:v3 chinakevinguo/sinatra:v5
```

```bash
#上传image到dockerHub上
docker push [OPTIONS] NAME[:TAG]

$ docker push chinakevinguo/docker-whale:latest
```

```bash
#删除镜像
docker rmi [OPTIONS] IMAGE [IMAGE...]
  -f 强制删除镜像(实际上是先删除了基于该镜像创建的容器之后再删除的该镜像)
$ docker rmi -f web1
```

#### Docker network management
* host模式：容器和宿主机共享network namespace
* container模式：容器和另外一个容器共享network namespace。kubernetes中的pod就是多容器共享一个network namespace
* none模式：容器有独立的network namespace，但没有对其进行任何的网络配置
* bridge模式：docker默认模式，容器通过一个网桥获取ip，以NAT的方式和外界通信
![](/images/docker-bridge-module.png)
**总结一下bridge网络就是：docker会在机器上自己维护一个网络，并通过docker0这个虚拟交换机和主机本身的网络连接在一起**

```bash
#docker网络管理
docker network COMMAND

COMMANDS:
  create       创建一个网络
  connect      链接容器到网络
  disconnect   将容器从网络断开
  inspect      显示网络的详细信息
  ls           列出网络
  rm           删除一个或者更多网络
```

```bash
#创建一个网络
docker network create [OPTIONS] YOURNETWORK
  -d 指定docker使用哪种模式的网络

$ docker network create -d bridge my-bridge-network
```

```bash
#将容器从网络断开
docker network disconnect [OPTIONS] NETWORK CONTAINER

$ docker network disconnect bridge web1
```

```bash
#将容器链接到网络
docker network connect [OPTIONS] NETWORK CONTAINER
  --ip   指定ip地址
  --link 添加连接到其他容器

$ docker network connect my-bridge-network web1
```

#### Docker container volume management
```bash
#docker volume 管理
docker volume COMMAND
  COMMANDS：
    create    创建卷组
    inspect   显示卷组详细信息
    ls        列出卷组
    rm        删除卷组
```

```bash
#为容器挂载卷组
$ docker run -d -P --name web -v /src/webapp:/webapp training/webapp python app.py
```
这个命令是将宿主机上的/src/webapp挂载到容器的/webapp，如果/webapp存在，则/src/webapp会覆盖挂载，但是不会删除以前的内容，取消此次挂载后，以前的内容又可以访问了。
容器内部的目录必须指定绝对路径，而宿主机上的目录即可以是绝对路径，也可以仅仅是一个名字或者什么都不写，如果只是一个名字$NAME，则会默认挂载到/var/lib/docker/volumes/$NAME/_data，如果什么都不写，则会挂载到/var/lib/docker/volumes/下的一个随机生成的字符串下。
```bash
#不指定名字，默认挂载
$ docker run -d -P --name web -v /webapp training/webapp python app.py

$ docker inspect web
"Name": "fac362...80535",
"Source": "/var/lib/docker/volumes/fac362...80535/_data",
"Destination": "/webapp"
```

```bash
#仅仅指定名字
$ docker run -d -P --name web -v webapp:/webapp training/webapp python app.py

$ docker inspect web
"Name": "webapp",
"Source": "/var/lib/docker/volumes/webapp/_data",
"Destination": "/webapp"
```

```bash
#指定绝对路径
$ docker run -d -P --name web -v /src/webapp:/webapp training/webapp python app.py

$ docker inspect web
"Source": "/src/webapp",
"Destination": "/webapp",
```

```bash
#挂载共享卷组作为数据卷
$ docker run -d -P --volume-driver=flocker -v my-named-volume:/webapp --name web training/webapp python app.py

#先创建共享卷组，然后再进行挂载
$ docker volume create -d flocker -o size-20GB my-named-volume
$ docker run -d -P -v my-named-volume:/webapp --name web training/webapp python app.py
```

```bash
#将容器作为卷组挂载进容器

#创建一个卷组容器
$ docker create -v /dbdata --name dbstore training/postgres /bin/true

#使用--volumes-from 将dbdata挂载进其他容器
$ docker run -d --volumes-from dbstore --name db1 training/postgres
$ docker run -d --volumes-from dbstore --name db2 training/postgres
```

#### Backup，restore，migrate data volumes
```bash
#将卷组容器挂载到对应的容器之后，进行卷组数据备份
$ docker run --rm --volumes-from dbstore -v $(pwd):/backup ubuntu tar cvf /backup/backup.tar /dbdata
```

```bash
#数据恢复
$ docker run -v /dbdata --name dbstore2 ubuntu /bin/bash
$ docker run --rm --volumes-from dbstore2 -v $(pwd):/backup ubuntu bash -c "cd /dbdata && tar xvf /backup/backup.tar --strip 1"
```

```bash
#在使用--rm的时候，会删除匿名卷组，而不会删除定义好的命名卷组
$ docker run --rm -v /foo -v awesome:/bar busybox top
```
**所谓匿名卷组：即那些指定了路径和那些没有指定命名的挂载项目**
