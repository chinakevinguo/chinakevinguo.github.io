---
layout: post
title: Docker基础-Dockerfile常用指令
categories: [docker]
description: docker Dockerfile常用指令集
keywords: docker
---
# Dockerfile reference

## Format
Dockerfile的指令是不区分大小写的，然而，通常我们约定俗成的都使用大写，为了与Dockerfile中的参数区分开。
```bash
#Comment
INSTRUCTION arguments
指令        参数
```
Dockerfile的指令在Dockerfile中按照顺序执行，第一条必须是`FROM`，指定你要构建的image的base image。
在Dockerfile中以#开头的为注释行，而在其他位置的#通常作为一个参数，比如
```bash
#Comment
RUN echo "we are running some # of cool things"
```
<!--more-->
#### Parser directives
```bash
#指定转义字符是什么，默认情况下的转义字符是反斜杠，但是，有时候，我们需要用转义字符来表示windows里面的文件路径分隔符，那么这个时候，我就需要用别的方式来表示转义字符了
# escape=`
```
Parser directives，指令解释器，解释某个指令在这个dockerfile中表示什么意思，默认情况下，反斜杠在windows中表示路径分隔符，然而如果在dockerfile中使用反斜杠，则会认为是转义符，那么这个时候，就需要重新指定一个转义符，将某个字符串转换成反斜杠，而默认的反斜杠就可以用来作为路径分隔符

**注意**
1.解释器指令必须在dockefile的第一行，放在别的地方会被认为是注释
2.解释器不支持单行连续换行
3.必须为正确的解释器指令

支持指令解释器的有：
escape


#### Environment replacement
Dockerfile中的如下指令内容支持以变量的形式呈现，同样也可以在变量前面加转义符进行转义，Dockerfile中的变量由ENV定义
#### .dockerignore file
`.dockerignore`用来忽略上下文目录中包含的一些image用不到的文件，它们不会传送到docker daemon。规则使用go语言的匹配语法。如：
```bash
$ cat .dockerignore
.git
tmp*
```
#### FROM
```bash
FROM <image>
or
#tag和digest是可选项
FROM <image>:<tag>
FROM <image>@<digest>
```
在Dockerfile中第一条非注释INSTRUCTION一定是`FROM`，它决定了以哪一个镜像作为基准，<image>首选本地是否存在，如果不存在则会从公共仓库下载（当然也可以使用私有仓库的格式）
#### MAINTAINER
```bash
MAINTAINER <name>
```
`MAINTAINER` 设定构建该镜像的作者的个人信息，包括姓名，邮箱等
#### RUN
```bash
RUN <command>
or
RUN ["executable","param1","param2"]
```
`RUN`指令会在当前镜像的每个新层的顶部执行命令，每个`RUN`指令运行之后都会生成一个新的层，生成的新层会被提交到image,然后在Dockerfile中定义的下一步所用到
上面写的`RUN`有两种格式

shell格式，相当于执行/bin/sh -c "<command>"
```bash
RUN apt-get install vim -y
```
exec格式，不会触发shell，主要是为了方便在没有bash的镜像中执行，而且可以避免错误的解析命令字符串：
```bash
RUN ["apt-get","install","vim","-y"]
or
RUN ["/bin/bash","-c","apt-get install vim -y"] 与shell风格相同
```

#### CMD
```bash
CMD ["executable","param1","param2"] exec格式
CMD ["param1","param2"] 作为ENTRYPOINT的默认参数
CMD command param1 param2 shell格式
```
一个Dockerfile中只能有一个`CMD`，如果有多个，只有最后一个生效。`CMD`指令的主要功能是在build完成后，为了给`docker run`启动到容器的时候提供默认命令或者参数，这些默认值可以包含任何可执行的命令，也可只是参数(**只是参数的时候可执行的命令就必须提前在`ENTRYPOINT`中指定**)

它与`ENTRYPOINT`的功能极为相似，区别在于如果使用`docker run`启动容器的时候指定了命令或者，那么Dockerfile中指定的`CMD`命令会被覆盖，而`ENTRYPOINT`则不会覆盖，只会把容器名后面的所有内容都当成参数传递给`ENTRYPOINT`指定的命令。另外`CMD`还可以单独作为`ENTRYPOINT`的所接命令的可选参数

`CMD`与`RUN`的区别在于，`RUN`是在`build`成镜像时运行的，先于`CMD`和`ENTRYPOINT`的，`CMD`会在每次启动容器的时候运行，而`RUN`只在创建镜像的时候执行一次，固话在image中

同样exec格式，不会触发shell，所以`$HOME`这样的变量无法使用

举例1：
```bash
Dockerfile:
    CMD ["echo","CMD_args"]
运行
    docker run <image>
结果
    输出 CMD_args
运行
    docker run <image> echo run_args
结果
    输出 run_args
```
默认会输出`CMD_args`，而在运行是输入`echo run_args`，则会输出`run_args`，因为新输入的命令覆盖了`CMD`

举例2：
```bash
Dockerfile:
    ENTRYPOINT ["echo","ENTRYPOINT_args"]
运行
    docker run <image>
结果
    输出 ENTRYPOINT_args
运行
    docker run <image> echo run_args
结果
    输出 ENTRYPOINT_args echo run_args
```
默认会输出`ENTRYPOINT_args`,如果输入`echo run_args`,，则会输出`ENTRYPOINT_args echo run_args`，因为使用的`ENTRYPOINT`,所有`docker run`后面的内容都是`ENTRYPOINT`的参数

举例3：
```bash
Dockerfile:
    ENTRYPOINT ["echo"]
    CMD ["echo","CMD_args"]
运行
    docker run <image>
结果
    输出 echo CMD_args
运行
    docker run <image> hello world
结果
    输出 hello world
```
默认会输出`echo CMD_args`,如果输入`hello world`，则会输出`hello world`，因为输入的`hello world`覆盖了`CMD`,当`CMD`和`ENTRYPOINT`同时出现的时候，`CMD`的内容只能作为`ENTRYPOINT`的参数

#### ENTRYPOINT
```bash
ENTRYPOINT ["executable","param1","param2"] exec格式，首选
ENTRYPOINT command param1 param2 shell格式
```
ENTRYPOINT 有两种写法，第二种(shell form)会屏蔽掉docker run时后面加的命令和CMD里的参数。
一个Dockerfile中只能有一个`ENTRYPOINT`，如果有多个，只有最后一个生效。`ENTRYPOINT`命令设置在容器启动时执行的命令
使用exec格式，在`docker run <image>`后的所有参数，都会追加到`ENTRYPOINT`之后，并且会覆盖所有`CMD`指定的参数。当然可以在`run`时使用`--entrypoint`来覆盖`ENTRYPOINT`指令
使用shell格式，`ENTRYPOINT`相当于执行`/bin/sh -c <command..>`，这种格式会忽略`docker run`和`CMD`的所有参数

同样exec格式，不会触发shell，所以像`$HOME`这样的环境变量是无法使用的

举例1：
```bash
Dockerfile:
    FROM ubuntu
    ENTRYPOINT ["top","-b"]
    CMD ["-c"]
运行
    docker run -ti --rm --name test chinakevinguo/sinatra:v5
结果
    top -b -c
运行
    docker run -ti --rm --name test chinakevinguo/sinatra:v5 -H
结果
    top -b -H
```
可以看到`CMD`指定的参数`-c`已经被覆盖，变成了`docker run <image>`所指定的`-H`，而`ENTRYPOINT`，所指定的`-b`参数依然存在

#### LABEL
`LABEL`主要是给image添加元数据，加上一个标签,通常以`KEY=VALUE`的形式添加，要在VALUE中要包含空格， 可使用引号和反斜杠
```bash
LABEL com.example.vendor="Kevin Guo" version="1.0" description="一个image可能有不止一个label,docker建议将所有的label都组合在一个LABEL中"
```
当在Dockerfile中使用`LABEL`后,基于该镜像运行容器，使用`docker inspect`可看到所有你打好的标签label
```bash
Labels: {
                "build-date": "20161102",
                "description": "this text illustrates that label-values can span multiple lines.",
                "license": "GPLv2",
                "name": "centos-test",
                "vendor": "KevinGuo",
                "version": "1.0"
            }

```

#### EXPOSE
```bash
EXPOSE <port> [<port>...]
```
EXPOSE指令告诉容器在运行时要监听的端口，但是这个端口只是用于多个容器之间通行用的(links),外面的host是无法访问的。要把容器端口暴露给外面的主机，在启动容器时使用-p/-P选项。
示例：
```bash
Dockerfile:
EXPOSE 8000 80 90
运行：
  docker run -d -P --name web chinakevinguo/httpd
结果：
  0.0.0.0:32775->80/tcp, 0.0.0.0:32774->90/tcp, 0.0.0.0:32773->8000/tcp
```
可以看到我在Dockerfile中指定要监听的端口都监听了，而且我使用`-P`选项，将这些被监听的端口都暴露出来了

#### ENV
使用ENV设置环境变量，保持环境一致，另外在Dockerfile同一行中EVN环境变量是保持不替换的，环境变量替换会在下一行中实现
```bash
ENV <key> <value>
ENV abc=hello
ENV abc=bye def=$abc
ENV ghi=$abc
#这个时候def=hello，而ghi=bye
```
设置了后，后续的RUN命令都可以使用，当运行生成的镜像时这些环境变量依然有效，如果需要在运行时更改这些环境变量可以在运行docker run时添加-env <key>=<value>参数来修改

#### ADD
```bash
ADD <src>... <dest>
or
ADD ["<src>",... "dest"] 路径包含空格的话，就需要这种格式
```
将文件`<src>`拷贝到container的文件系统对应的路径`<dest>`下。
`<src>`可以是文件、文件夹、URL,对于文件和文件夹`<src>`必须是在Dockerfile的相对路径下，即只能是Dockerfile的相对路径且不能使用类似`../path/`的方式
`<dest>`只能是容器中的绝对路径，如果路径不存在则会自动级联创建，根据你的需要决定`<dest>`是否需要反斜杠`/`，使用`/`结尾则是目录，否则就是文件

示例：
```bash
支持模糊匹配
ADD home* /mydir/   # adds all files starting with "hom"
ADD home?.txt /mydir/ # ? is replaced with any aingle character

ADD requirements.txt /tmp/
RUN pip install /tmp/requirements.txt
ADD . /tmp/
```

另外`ADD`还支持从远程URL获取文件，但是官方强烈反对这样做，建议使用`wget`或`curl`代替
`ADD` 还支持自动解压tar文件，这是`ADD`和`COPY`最大的区别

#### COPY
```bash
ADD <src>... <dest>
or
ADD ["<src>",... "dest"] 路径包含空格的话，就需要这种格式
```
`COPY`的语法与功能与`ADD`相同，只是不支持上面讲到的`<src>`
是远程URL、自动解压这两个特性，但是[Best Practices for Writing Dockerfiles](https://docs.docker.com/engine/userguide/eng-image/dockerfile_best-practices/)建议**尽量使用**`COPY`,并使用 `RUN`与`COPY`组合来代替`ADD`,建议只有在复制tar文件的时候使用`ADD`

#### VOLUME
```bash
VOLUME ["/data1","/data2"]
```
VOLUME指令用来在容器中设置一个挂载点，可以用来让其他容器挂载以实现数据共享或对容器数据的备份、恢复或迁移,请参考文章[Manage data in containers](https://docs.docker.com/engine/tutorials/dockervolumes/#mount-a-host-directory-as-a-data-volume)
示例：
```bash
FROM ubuntu
RUN mkdir /myvol
RUN echo "hello world" > /myvol/greeting
VOLUME /myvol
```
这个Dockerfile会导致这个image创建一个挂载点`/myvol`，然后将`greeting`文件copy到新建的卷组中

#### WORKDIR
```bash
WORKDIR /path/to/workdir
```
`WORKDIR`指令用于设置`Dockerfile`中`RUN`、`CMD`、`COPY`、`ADD`和`ENTRYPOINT`指令执行命令的工作目录(默认为/目录)，该指令在`Dockerfile`文件中可以出现多次，如果使用相对路径则为相对于`WORKDIR`上一次的值，例如：`WORKDDIR /a`,`WORKDIR b/`,`RUN pwd` 最终输出的当前目录是`/a/b`
`WORKDIR`还能够解析通过ENV指定的环境变量
```bash
ENV DIRPATH /path
WORKDIR $DIRPATH/$DIRNAME
RUN pwd
```

#### USER
```bash
USER daemon
```
`USER`为运行镜像时或者任何接下来的`RUN`，`CMD`,`ENTRYPOINT`等指令指定运行用户名或UID

#### ARG
```bash
ARG <name>[=<default value>]
```
ARG 指令定义一个变量，用户可以在构建的时候使用`docker build`命令，并使用--build-arg <varname>=<value>标志传递给构建器，并且`ARG`定义的变量只有在构建image的时候有效，构建完成后就会消失，而`ENV`指定的环境变量则会持续存在
示例：
```bash
FROM busybox
ARG user1
ARG buildno
```
如果`ARG`没有默认值，在构建是就必须指定值，否则会报错
```bash
FROM busybox
ARG user1=someuser
ARG buildno=1
```
如果`ARG`有默认值，在构建时没有指定值则使用默认值，在构建时指定了值，则使用指定的值

`ARG`变量从在Dockerfile中定义的时候就开始生效，比如，看如下的Dockerfile：
```bash
FROM busybox
USER ${user:-some_user}
ARG user
USER $user
```
```bash
$ docker build --build-arg user=what_user -t chinakevinguo/web .
```
通过`docker inspect image`查看
```bash
"User": "what_user"
```
第2行的user并没有变量值，所以是默认指定的some_user,而第4行的USER的值则是从ARG传递进来的what_user

```bash
FROM ubuntu
ARG CONT_IMG_VER
ENV CONT_IMG_VER v1.0.0
RUN echo $CONT_IMG_VER
```
使用`ENV`的环境变量总是会覆盖`ARG`的环境变量，所以我们可以使用`ARG`来传递可变参数，然后通过`ENV`来永久保存到IMAGE中
docker中有一组与定义的`ARG`变量，你可以在Dockerfile中使用相应的`ARG`指令
* HTTP_PROXY
* http_proxy
* HTTPS_PROXY
* https_proxy
* FTP_PROXY
* ftp_proxy
* NO_PROXY
* no_proxy

#### ONBUILD
`ONBUILD`指令用来设置一些触发指令，用于在当该镜像被作为基础镜像来创建其他镜像时(也就是`Dockerfile`中的`FROM`为当前镜像时)执行一些操作，`ONBUILD`中定义的指令会在用于生成器他镜像的`Dockerfile`文件的`FROM`指令之后被执行，上述介绍的任何一个指令都可以用于`ONBUILD`指令(除了`FROM`和`MAINTAINER`)，可以用来执行一些因环境变化而引起的操作，使镜像更加通用。
**注意：**
  1.`ONBUILD`中定义的指令在当前镜像的build中不会被执行
  2.可以通过`docker inspect <image>`命令，查看输出的ONBUILD键来查看某个镜像ONBUILD指令指定的内容
  3.`ONBUILD`指令会在下游镜像被触发执行，执行顺序会按`ONBUILD`定义的先后顺序执行
  4.引用`ONBUILD`的镜像创建完成后将会清除所有引用的`ONBUILD`指令
  5.ONBUILD指令不允许嵌套，例如：`ONBUILD ONBUILD ADD ./data` 是不允许的
  6.ONBUILD指令不会触发`FROM`或`MAINTAINER`指令

例如，Dockerfile使用如下内容创建了镜像image-A：
```bash
[...]
ONBUILD ADD . /app/src
ONBUILD RUN /usr/local/bin/python-build --dir /app/src
[...]
```
如果基于image-A创建新镜像时，新的Dockerfile中使用`FROM image-A`指定基础镜像时，会自动执行`ONBUILD`指令内容，等价于在后面添加了两条指令
```bash
FROM image-A
#Automatically run the following
ADD . /app/src
RUN /usr/local/bin/python-build --dir /app/src
```

#### STOPSIGNAL
```bash
STOPSIGNAL signal
```
`STOPSIGNAL`指令用来设置停止容器时发送什么系统调用信号给容器，这个信号必须是内核系统调用表中合法的数，例如9，或者是`SIGNAME`格式的信号名称，例如`SIGKILL`

#### HEALTHCHECK
```bash
HEALTHCHECK [OPTIONS] CMD command (通过在容器内运行命令来对容器进行健康检查)
or
HEALTHCHECK NONE (禁用所有从基础镜像继承的健康检查)
```
`HEALTHCHECK`指令用来告诉Docker怎样去测试一个容器是否还在工作，这可以检测诸如，web服务器卡住了无法处理新的连接，但是服务的进程仍然在运行等情况
当容器指定了`HEALTHCHECK`时，其除了正常的状态外，还具有健康状态，这个指定的healthckeck状态是初始状态，每当健康检查通过，就认定这个容器是健康的（无论之前的状态如何），当发生故障后，它就变得不健康了

可在`CMD`前添加的可选项：
* --interval=时长[默认30s] 每隔多久检测一次
* --timeout=时长[默认30s]  如果在单次检测的时长超过设定值
* --retries=次数[默认3次]   重复检查多少次后才被视为不健康

另外：在一个Dockerfile中只能有一个`HEALTHCKECK`，如果存在多个，则最后一个生效
`CMD`之后的命令可以是shell命令(HEALTHCHECK CMD /bin/check-running)，也可以是exec格式(["/bin/sh","check-running"])
命令的退出状态表示容器的运行状态，可能值为：
0：success - the container is healthy and ready for use
1：unhealthy - the container is not working correctly
2：reserved - do not use this exit code

示例：
```bash
# 每隔5分钟检测一次web服务器，如果超过3秒无响应，则视为不健康
HEALTHCHECK --interval=5m --timeout=3s CMD curl -f http://localhost/ || exit 1
```

#### SHELL
```bash
SHELL ["executable","parameters"]
```
`SHELL`指令用于覆盖使用默认shell格式的shell命令，在linux上默认的shell是["/bin/sh","-c"]，在windows上是["cmd","/S","/C"]，`SHELL`指令在dockerfile中必须以JSON的格式来写
`SHELL`指令在windows上尤其有用，因为windows上的`powershell`和`cmd`这两种shell
`SHELL`指令可以添加多次，买个`SHELL`指令都会覆盖前面的`SHELL`指令，并影响后面的所有指令，例如：
```bash
FROM windowsservercore

# 默认执行使用cmd /S /C echo
RUN echo default

# 默认执行使用cmd /S /C powershell -command Write-Host
RUN powershell -command Write-Host default

# 使用SHELL 指定使用的shell是powershell
SHELL ["powershell", "-command"]
RUN Write-Host hello

# 使用SHELL 指定使用的shell是cmd /S /C
SHELL ["cmd", "/S", "/C"]
RUN echo hello
```
**注意**
当使用的`SHELL`格式发生变化，那么诸如:`RUN`,`CMD`,`ENTRYPOINT`等指令调用命令的方式也会发生变化，比如：
```bash
...
# Dockerfile 中定义：
RUN powershell -command Execute-MyCmdlet -param1 "c:\foo.txt"
# Docker实际调用的命令是`cmd /S /C powershell -command Execute-MyCmdlet -param1 "c:\foo.txt"`
```
然而上述方法效率很低，因为首先，有一个不必要的`cmd.exe`被调用，其次，shell中的每个RUN都需要指定一个额外的`powershell -command`
更高效的做法是使用`SHELL`指令和shell格式来提供更自然的语法：
```bash
# escape=` #这是指令解释器，将`解释成转义符
FROM windowsservercore
SHELL ["powershell","-command"]
RUN New-Item -ItemType Directory C:\Example
ADD Execute-MyCmdlet.ps1 c:\example\
RUN C:\example\Execute-MyCmdlet -sample 'hello world'
```

#### Dockerfile examples
下面是一些Dockerfile的例子，更多内容请参考[Dockerization examples](https://docs.docker.com/engine/examples/)
```bash
# Nginx
#
# VERSION               0.0.1

FROM      ubuntu
MAINTAINER Victor Vieux <victor@docker.com>

LABEL Description="This image is used to start the foobar executable" Vendor="ACME Products" Version="1.0"
RUN apt-get update && apt-get install -y inotify-tools nginx apache2 openssh-server
```

```bash
# Firefox over VNC
#
# VERSION               0.3

FROM ubuntu

# Install vnc, xvfb in order to create a 'fake' display and firefox
RUN apt-get update && apt-get install -y x11vnc xvfb firefox
RUN mkdir ~/.vnc
# Setup a password
RUN x11vnc -storepasswd 1234 ~/.vnc/passwd
# Autostart firefox (might not be the best way, but it does the trick)
RUN bash -c 'echo "firefox" >> /.bashrc'

EXPOSE 5900
CMD    ["x11vnc", "-forever", "-usepw", "-create"]
```

```bash
# Multiple images example
#
# VERSION               0.1

FROM ubuntu
RUN echo foo > bar
# Will output something like ===> 907ad6c2736f

FROM ubuntu
RUN echo moo > oink
# Will output something like ===> 695d7793cbe4

# You᾿ll now have two images, 907ad6c2736f with /bar, and 695d7793cbe4 with
# /oink.
```
