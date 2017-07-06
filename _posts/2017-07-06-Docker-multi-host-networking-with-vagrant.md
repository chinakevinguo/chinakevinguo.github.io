---
layout: post
title: Docker基础-docker multi-host网络测试
categories: [docker]
description: docker multi-host网络之overlay网络测试
keywords: docker
---
这篇只是上一篇主要介绍了，如何使用`vagrant`来创建虚拟机，进行外部`key-value`存储overlay网络的测试。

<!--more-->
# 使用vagrant创建实验环境
什么是`vagrant`?请参考[vagrant官网](https://www.vagrantup.com/)
## 创建实验环境
安装`virtualbox`,`vagrant`,`git-bash`
1.打开git-bash,新建一个vagrant工作的目录
```bash
$ mkdir docker-multi-host
```
2.初始化一个vagrant配置文件
```bash
$ cd docker-multi-host
$ vagrant init
A `Vagrantfile` has been placed in this directory. You are now
ready to `vagrant up` your first virtual environment! Please read
the comments in the Vagrantfile as well as documentation on
`vagrantup.com` for more information on using Vagrant
```
3.你会发现目录下多了个`Vagrantfile`文件，编辑这个文件，替换成如下内容：
```ruby
## files: Vagrantfile
# -*- mode: ruby -*-
# vi: set ft=ruby :
hosts = {
  "node01" => "192.168.1.1",
  "node02" => "192.168.1.2",
  "node03" => "192.168.1.3",
  "node04" => "192.168.1.4",
  "node05" => "192.168.1.5"
}

Vagrant.configure("2") do |config|
  config.vm.box = "bento/centos-7.2"
  config.vm.box_url = "./vagrant-centos-7.2.box"
  config.vm.provider "virtualbox" do |v|
    v.customize ["modifyvm",:id,"--memory",512]
  end
  config.vm.define "my-keystore" do |machine|
    machine.vm.network :private_network, ip: "192.168.1.100"
    machine.vm.hostname = "my-keystore"
    machine.vm.synced_folder ".","/vagrant", disabled: true
    machine.vm.provision "shell",
    run: "always",
    inline: "sudo ifup enp0s8"
  end
  hosts.each do |name, ip|
    config.vm.define name do |nodes|
      nodes.vm.hostname = name
      nodes.vm.network :private_network, ip: ip
      nodes.vm.provision "shell",
        run: "always",
        inline: "sudo ifup enp0s8"
      end
    end
  end

```
> **NOTE:** 该配置文件是ruby语法，具体可参考[vagrant官网](https://www.vagrantup.com)

4.使用`vagrant up`启动
```bash
$ vagrant up
Bringing machine 'my-keystore' up with 'virtualbox' provider...
```
5.验证刚刚启动的虚拟机的状态
```bash
$ vagrant status
Current machine states:

my-keystore               running (virtualbox)
node01                    running (virtualbox)
node02                    running (virtualbox)
node03                    running (virtualbox)
node04                    running (virtualbox)
node05                    running (virtualbox)

```
6.使用ssh登录VM，两种方法：
* 方法一：
  使用`vagrant ssh <vm-name>`直接在git-bash里登录
  $ vagrant ssh my-keystore
  Last login: Wed Jan  4 06:26:23 2017 from 10.0.2.2
  [vagrant@my-keystore ~]$

* 方法二：
  使用ssh客户端工具登录，如XSHELL等等
> 至此，实验所需的环境，我们已经搭建完毕


## 配置key-value store
overlay网络依赖key-value存储.key-value存储保存着一些关于网络的状态信息，包括，发现，网络，终端，IP地址等等.Docker 支持Consul,Etcd,ZooKepper等key-value存储，这里使用的是Consul.
1.进入`my-keystore`虚拟机，配置安装
```bash
# 系统更新
$ yum update -y
#安装时间同步
$ yum install ntp -y
#配置hosts文件
$ cat /etc/hosts
192.168.1.100 my-keystore
192.168.1.1 node01
192.168.1.2 node02
192.168.1.3 node03
192.168.1.4 node04
192.168.1.5 node05
#添加docker repo
$ tee /etc/yum.repos.d/docker.repo <<-'EOF'
[dockerrepo]
name=Docker Repository
baseurl=https://yum.dockerproject.org/repo/main/centos/7/
enabled=1
gpgcheck=1
gpgkey=https://yum.dockerproject.org/gpg
EOF
# 安装docker
$ yum install docker-engine -y
```
2.在`my-keystore`虚拟机中运行一个consul容器
```bash
$  docker run -d \
     -p "8500:8500" \
     -h "consul" \
     progrium/consul -server -bootstrap
```
3.我们可以看到在mh-keystore里面已经有了一个运行着consul的容器,并监听到8500端口
```bash
[root@my-keystore ~]# docker ps
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS                                                                            NAMES
3cdee7709575        progrium/consul     "/bin/start -server -"   3 hours ago         Up 2 seconds        53/tcp, 53/udp, 8300-8302/tcp, 8400/tcp, 8301-8302/udp, 0.0.0.0:8500->8500/tcp   angry_gates
```

## 配置其他所有swarm节点
1.进入所有swarm节点虚拟机，配置安装
```bash
# 系统更新
$ yum update -y
#安装时间同步
$ yum install ntp -y
#配置hosts文件
$ cat /etc/hosts
192.168.1.100 my-keystore
192.168.1.1 node01
192.168.1.2 node02
192.168.1.3 node03
192.168.1.4 node04
192.168.1.5 node05
#添加docker repo
$ tee /etc/yum.repos.d/docker.repo <<-'EOF'
[dockerrepo]
name=Docker Repository
baseurl=https://yum.dockerproject.org/repo/main/centos/7/
enabled=1
gpgcheck=1
gpgkey=https://yum.dockerproject.org/gpg
EOF
# 安装docker
$ yum install docker-engine -y
```
2.在所有swarm节点上配置docker daemon的启动配置文件
```bash
vi /usr/lib/systemed/system/docker.service
ExecStart=/usr/bin/dockerd -H tcp://0.0.0.0:2375 -H unix:///var/run/docker.sock --cluster-store consul://my-keystore:8500 --cluster-advertise enp0s8:2375
```
3.配置完成后，使用`docker info`查看
```bash
[root@node01 ~]# docker info
Containers: 0
 Running: 0
 Paused: 0
 Stopped: 0
Images: 0
Server Version: 1.12.5
Storage Driver: devicemapper
...
Cluster Store: consul://my-keystore:8500
Cluster Advertise: 192.168.1.1:2375
Insecure Registries:
 127.0.0.0/8
```
4.创建一个`overlay`网络
```bash
$  docker network create --driver overlay --subnet=10.0.9.0/24 my-net
```
5.去所有节点上看看，是否已经有了这个`my-net`网络
```bash
#进入node01
[root@node01 ~]# docker network ls
NETWORK ID          NAME                DRIVER              SCOPE
0cacaf83d82f        bridge              bridge              local               
fc2a7860f8a1        host                host                local               
3fc32842e291        my-net              overlay             global              
ebf3c8e333e9        none                null                local

#进入node02
[root@node02 ~]# docker network ls
NETWORK ID          NAME                DRIVER              SCOPE
e3ed743a143c        bridge              bridge              local               
1274cd7fe49f        host                host                local               
3fc32842e291        my-net              overlay             global              
03724cede8c1        none                null                local
```
可以看到`my-net`网络在两个节点上都生效了
6.这时候也可以直接去consul上查看
![consul-key-value](/images/posts/consul-key-value.png)
可以看到我们已经在consul上存储了两个网络(其中有一个网络是我以前创建的)

## 测试.在`my-net`网络上运行应用试试
1.在`node01`上运行一个nginx web应用
```bash
$ docker run -itd --name=web --network=my-net --env="constraint:node==node01" nginx
```
2.在`node02`上运行一个busybox容器，来获取Nginx服务器主页上的内容
```bash
$ docker run -it --rm --network=my-net --env="constraint:node==node02" busybox wget -O- http://web
Connecting to web (10.0.9.2:80)
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
    body {
        width: 35em;
        margin: 0 auto;
        font-family: Tahoma, Verdana, Arial, sans-serif;
    }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
-                    100% |*******************************|   612   0:00:00 ETA

```
可以看到是通过my-net网络进行访问的

> **PS** 这里我们可以对比上一篇[Get started with multi-host networking](http://www.oo3p.com/2016/12/30/multi-host-networking/)，我们发现，在使用`key-value`存储的时候，只要`docker engine`指定了`key-value`存储，就能获取到网络，而使用`swarm mode`的时候，则必须是在`swarm manager`管理下的节点才能获取网络。
