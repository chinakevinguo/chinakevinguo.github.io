---
layout: post
title: Docker基础-docker容器之间如何通过link进行通信
categories: [docker]
description: docker容器之间通过link通信
keywords: docker
---
### 传统容器链接

本章节主要介绍了在我们安装Docker的时候，自动创建的docker 默认`bridge`网络中的传统容器link。
<!--more-->
在Docker network 功能之前，你可以使用docker的link功能来允许容器彼此发现，将一个容器的信息安全的传输到另一个容器。随着docker network功能的引入，您仍然可以创建link。但是link功能在默认的bridge网络和用户自定义网络上还是有所不同。

> **Warning:** `--link`是docker的一个传统功能。它以后可能会被移除。除非你确定你非要使用它，否则，我们建议你最好使用用户自定义网络。用户自定义网络唯一不支持的是，你可以使用--link来在容器之间共享环境变量。但是，你可以使用其他方式(如volumes)等方式来在容器中共享环境变量。


### 使用端口映射连接

```bash
$ docker run -d -P training/webapp python app.py
```
当容器运行的时候，使用`-P`选项，会将容器内部发布的端口，随机映射到书主机上的任意端口，这里可以看到，容器的5000端口，已经绑定到了宿主机上的32768端口上
```bash
$ docker ps -l
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS                     NAMES
0de1cf226109        training/webapp     "python app.py"     2 minutes ago       Up 2 minutes        0.0.0.0:32768->5000/tcp   stoic_lamport
```
你也可以使用`-p`选项，来指定绑定到宿主机的端口
```bash
$ docker run -d -p 6666:5000 training/webapp python app.py
```

你甚至可以指定绑定到宿主机的端口范围
```bash
$ docker run -d -p 8000-9000:5000 training/webapp python app.py
```

默认情况下，`-p`标志是绑定到所有的接口。你可以绑定到指定的接口的指定端口，如localhost
```bash
$ docker run -d -p 127.0.0.1:8000-9000:5000 training/webapp python app.py
```

当然，你也可以绑定到指定接口的随机端口上
```bash
$ docker run -d -p 127.0.0.1::5000 training/webapp python app.py
```

你还可以绑定UDP端口
```bash
$ docker run -d -p 127.0.0.1:80:5000/udp training/webapp python app.py
```

我们可以使用`docker port`来查看绑定的信息
```bash
$ docker port nostalgic_morse 5000

127.0.0.1:49155
```

### 使用link连接

本节主要是讲解的传统的link功能，关于用户自定义网络的link，可参考[docker network command](https://kevinguo.me/2017/07/06/Docker-network-command/)
端口映射不是容器彼此连接的唯一方式，你还可以将多个容器连接到一起，并将信息从一个发动到另一个。当容器被连接后，关于源容器的信息，可以发动到接受容器，这允许接收者看到源容器的数据。

#### 命名的重要性

为了建立链接，docker依赖于你的容器名。你可以看到，当你创建容器的时候，它会创建一个默认的容器名，实际上，你可以自定义容器名，给容器命名有两个好处：
1.好记
2.方便连接

我们可以使用`--name`来进行容器命名
```bash
$ docker run -d -P --name web training/webapp python app.py
```

### 连接之间的通信

link 允许容器之间互相发现 ，并将一个容器的信息安全的传送到另外一个容器。在设置link时，会在两个容器间创建一个父子关系，其中父容器可以访问子容器内部的某些信息。

首先，我们来创建一个数据库的容器
```bash
$ docker run -d --name db training/postgres
```

然后，创建一个web容器，并且将它连接到db容器
```bash
$ docker run -d -P --name web --link db:db training/webapp python app.py
```
这将会把新的web容器和之前的db容器连接起来，web容器可以访问db容器的信息

我们知道link允许子容器向父容器提供自身的信息。在我们上面的例子中，父容器web可以访问子容器db提供的数据库信息，为此，我们在创建子容器的时候，不需要在容器外部暴露任何端口，你会发现，上面在创建db容器的时候没有指定`-p`或`-P`,这样做的好处就是避免了暴露子容器（数据库）端口的风险。

docker以两种方式将子容器的连接信息暴露给父容器：
* 环境变量
* 更新/etc/hosts配置文件

#### 环境变量

当你链接容器的时候docker会创建几个环境变量。docker会基于`--link`目标参数在目标容器中创建环境变量。它还将从子容器暴露所有来自docker主机的环境变量，包括：
* Dockerfile中指定的ENV
* 启动容器时通过`-e`,`--env`和`--env-file`选项指定的环境变量

> **Warning:** 来自docker的所有环境变量对于目标容器都是可用的，如果有敏感数据存储器中，这可能会有安全风险

Docker为`--link`后的每个目标容器设定了别名。例如，如果一个名叫web的新容器连接到一个名叫db的数据库容器(`--link db:webdb`)。那么docker会在`web`容器中创建一个`WEBDB_NAME=/web/webdb`的环境变量

Docker还会为源容器公开的每个端口定义一组环境变量。每个变量在表单中都有一个唯一的前缀：

`<name>_PORT_<port>_<protocol>`

前缀部分包含如下：

* 通过`--link`指定的别名
* 公开的端口
* tcp/udp协议

Docker使用这种前缀格式指定了三个不同的环境变量：

* `prefix_ADDR` IP，例：WEBDB_PORT_5432_TCP_ADDR=172.17.0.82
* `prefix_PORT` 端口，例：WEBDB_PORT_5432_TCP_PORT=5432
* `prefix_PROTO` 协议，例：WEBDB_PORT_5432_TCP_PROTO=tcp

如果容器公开了多个端口，那么每个端口都要定义一个变量集。这意味着，比如，如果一个容器公开了4个端口，那么我们就需要设置12个环境变量，每个端口三个变量(包含ip，端口，协议)

另外，docker会创建一个名叫`<alias>_PORT`的环境变量，这个环境变量是源容器的第一个公开端口。这个所谓的`第一`端口，被称之为最低编号的暴露端口。例如：`DB_PORT=tcp://172.17.0.82:5432`

最后，docker还会将源容器里那些来自docker主机的环境变量暴露为目标容器的变量，这些变量会在目标容器中被创建成 `<alias>_ENV_<name>` 的样子，变量的值来自于源容器

我们可以进入web容器，使用`env`命令看看这个目标容器里面的环境变量

```bash
$ docker exec -ti web env

DB_PORT=tcp://172.17.0.2:5432
DB_PORT_5432_TCP=tcp://172.17.0.2:5432
DB_PORT_5432_TCP_ADDR=172.17.0.2
DB_PORT_5432_TCP_PORT=5432
DB_PORT_5432_TCP_PROTO=tcp
DB_NAME=/web/db
```
你可以看到docker已经创建了一些列有关源db容器的环境变量，每个变量的前缀都是`DB_`,这个DB就是你在上面指定的别名。如果别名是db1,那么这个前缀就是`DB1_`.你可以使用这些环境变量来配置应用程序连接到db容器上的数据库，连接是安全的，只有目标容器web能够访问源容器db。

#### 有关docker环境变量的重要注意事项

与`/etc/hosts file`中的host条目不一样，如果重新启动源容器，不会自动更新存储在环境变量中的IP地址，我们建议使用/etc/hosts来解析连接容器的IP地址。

#### 更新/etc/hosts配置文件

除了环境变量，Docker还会将源容器的主机条目添加到/etc/hosts文件中：
```bash
$ docker exec -ti web cat /etc/hosts

172.17.0.3	80e0d81d8652
172.17.0.2	db 60bedbc24041 webdb
```
可以看到有两条主机条目。第一条是使用容器ID作为主机名的web容器，第二条是使用容器名和别名来作为主机名的db容器。除了你提供的别名之外，容器本身的容器名也会被添加到/etc/hosts，这两个主机条目你都可以ping通：
```bash
$ docker exec -ti web ping db
PING webdb (172.17.0.2) 56(84) bytes of data.
64 bytes from webdb (172.17.0.2): icmp_seq=1 ttl=64 time=0.044 ms

$ docker exec -ti web ping webdb
PING webdb (172.17.0.2) 56(84) bytes of data.
64 bytes from webdb (172.17.0.2): icmp_seq=1 ttl=64 time=0.044 ms
```
> **Note:** 我们可以将当个源容器连接到多个目标容器

如果我们重启了源容器，目标容器的/etc/hosts会自动更新到新的IP地址
```bash
$ docker restart db
$ docker exec -ti web cat /etc/hosts
172.17.0.5	webdb 60bedbc24041 db
172.17.0.3	80e0d81d8652
```
而我们来看看目标容器web的env，发现环境变量并没有更新
```bash
$ docker exec -ti web env

DB_PORT=tcp://172.17.0.2:5432
DB_PORT_5432_TCP=tcp://172.17.0.2:5432
DB_PORT_5432_TCP_ADDR=172.17.0.2
DB_PORT_5432_TCP_PORT=5432
DB_PORT_5432_TCP_PROTO=tcp
DB_NAME=/web/db
DB_ENV_PG_VERSION=9.3
```
