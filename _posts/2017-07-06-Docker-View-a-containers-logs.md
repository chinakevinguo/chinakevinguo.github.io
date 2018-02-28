---
layout: post
title: Docker基础-docker容器日志处理方式
categories: [docker]
description: docker 容器日志处理方式
keywords: docker
---

`docker logs`显示了正在运行的容器的日志信息。默认情况下，`docker logs`显示命令的输出，就好象在终端中交互运行命令时一样。UNIX和Linux命令通常在运行时打开3个I/O流，成为STDIN,STDOUT和STDERR。STDIN为标准输入，其可以包括来自键盘的输入或来自另一个命令的输入。STDOUT为标准输出，STDERR为错误输出。默认情况下，docker日志显示STDOUT和STDERR。更多关于Linux上的I/O信息，可参考[ Linux Documentation Project article on I/O redirection](http://www.tldp.org/LDP/abs/html/io-redirection.html)

<!--more-->
有时候，如果我们采取了下面的操作，那么可能在终端上使用docker logs就无法获取到日志了：
* 如果使用了 [logging driver](https://docs.docker.com/engine/admin/logging/overview/)，将日志保存到文件，外部主机，数据库，或者其他后端日志服务器。
* 如果运行一个非交互式过程的镜像[如Web服务器或数据库]，那么该应用程序可能会将其输出到日志文件，而不会以STDOUT或STDERR的方式输出来。

在第一种情况下，您的日志以其他方式处理，您可以选择不使用docker日志。
在第二种情况下，nginx image显示一种方法，httpd image有另一种方法。

官方nginx image创建了一个从/var/log/nginx/access.log 到 /dev/stdout的符号链接，并创建了从/var/log/nginx/error.log 到/dev/stderr的另一个连接。可参考[Nginx Dockerfile](https://github.com/nginxinc/docker-nginx/blob/8921999083def7ba43a06fabd5f80e4406651353/mainline/jessie/Dockerfile#L21-L23)

官方httpd更改了httpd应用程序的配置，将其正常输出直接写入/proc/self/fd/1(这是STDOUT)，并将错误输出写入/proc/self/fd/2(这是STDERR)。可参考[Apache Dockerfile](https://github.com/docker-library/httpd/blob/b13054c7de5c74bbaa6d595dbe38969e6d4f860c/2.2/Dockerfile#L72-L75)
