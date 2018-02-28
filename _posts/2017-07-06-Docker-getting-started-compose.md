---
layout: post
title: Docker基础-docker compose基础知识
categories: [docker]
description: docker compose基础
keywords: docker
---

这一章，我们构建一个在docker compose上运行的简单python web程序。该应用使用Flask框架并在redis中维护一个命中计数器。
<!--more-->

# 前提准备

确保你安装了docker engine和docker compose。不需要安装python或redis，这些都由docker image来提供。


# 步骤1：建立

1.建立项目目录
```bash
$ mkdir composetest
$ cd composetest
```
2.建一个`app.py`文件在你的项目目录下，并copy下面的内容
```python
from flask import Flask
from redis import Redis

app = Flask(__name__)
redis = Redis(host='redis', port=6379)

@app.route('/')
def hello():
    count = redis.incr('hits')
    return 'Hello World! I have been seen {} times.\n'.format(count)

if __name__ == "__main__":
    app.run(host="0.0.0.0", debug=True)
```
3.在项目目录下建另一个名叫`requirements.txt`的文件
```bash
Flask
redis
```
这些内容定义了应用的依赖

# 步骤2：创建Dockerfile

在这一步，我们编写一个用于构建docker image的Dockerfile。该image包含了python应用程序所需的所有依赖关系，包括python本身

1.在项目目录下面，新建一个Dockerfile文件，内容如下
```bash
FROM python:3.4-alpine
ADD . /code
WORKDIR /code
RUN pip install -r requirements.txt
CMD ["python", "app.py"]
```

该Dockerfile告诉docker：
* 从python 3.4构建一个image
* 将当前目录添加到image的`/code`
* 设置工作目录为`/code`
* 运行pip安装python依赖
* 设定默认的CMD命令

# 步骤3：在compose file中定义服务

1.建一个`docker-compose.yml`文件，内容如下：
```yaml
version: '2'
services:
  web:
    build: .
    ports:
     - "5000:5000"
    volumes:
     - .:/code
  redis:
    image: "redis:alpine"
```

该compose file定义了两个服务，`web`和`redis`:

web service:
* 使用由当前目录下的Dockerfile构建的image
* 暴露5000端口到宿主机的5000端口
* 挂载当前项目目录到`/code`目录

redis service：
* 直接使用从docker hub上拉取的image

# 步骤4：使用compose构建和运行你的应用

1.在你的项目目录，启动你的应用
```bash
$ docker-compose up
 Pulling image redis...
 Building web...
 Starting composetest_redis_1...
 Starting composetest_web_1...
 redis_1 | [8] 02 Jan 18:43:35.576 # Server started, Redis version 2.8.3
 web_1   |  * Running on http://0.0.0.0:5000/
 web_1   |  * Restarting with stat
```
compose将会构建web镜像，拉取redis镜像，然后启动服务
2.在浏览器中反问http://ip:5000，将会显示`Hello World! I have been seen 1 times.`,并且，每当你刷新一次页面，次数会加1.

# 步骤5：更新应用

因为我们是将代码挂载到容器的卷组里面的，所以，我们可以随意修改代码，而不需要重新构建image
1.改变`app.py`的代码
```bash
return 'Hello from Docker! I have been seen {} times.\n'.format(count)
```

2.在浏览器中刷新试试看
刷新后，发现结果已经变成你刚刚改的内容

# 步骤6：试试其他的命令

如果我们想我们的服务在后台运行，我们可以在`docker-compose`后指定`-d`选项，然后使用`docker-compose ps`查看当前运行的服务
```bash
$ docker-compose up -d
Starting composetest_redis_1...
Starting composetest_web_1...

$ docker-compose ps
Name                 Command            State       Ports
-------------------------------------------------------------------
composetest_redis_1   /usr/local/bin/run         Up
composetest_web_1     /bin/sh -c python app.py   Up      5000->5000/tcp
```

`docker-compose run`命令允许为服务运行一次性命令，这也是我们在前面提到过的compose的一个特性。例如，我们要查看web服务的环境变量：
```bash
$ docker-compose run web env
```

更多关于docker-compose的命令，可使用`docker-compose --help`来查看

如果你使用`docker-cmpose up -d`来启动，那么你可能需要使用下面的方式来停止服务
```bash
$ docker-compose stop
```

我们还可以把一切都关掉，用`down`命令停止并删除容器，网络或images和volumes，传递 `--volumes`也可以删除redis容器使用的数据卷
```bash
$ docker-compose down --volumes
```
