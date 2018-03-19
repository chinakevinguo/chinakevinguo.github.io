---
layout: post
title: Docker基础-docker compose安装
categories: [docker]
description: docker compose安装
keywords: docker
---

你可以在macOS，windows 和64位的linux系统上运行compose。在安装它之前，你需要先安装docker。

<!--more-->

步骤如下：
1.安装docker engine
2.docker toolbox安装包括了docker engine和compose，因此Mac和windows已经安装好了。其他系统需要进入下一步
3.进入[compose repository release page on Github](https://github.com/docker/compose/releases)
4.按照发行页中的说明，在你的终端下运行指定的`curl`命令
> 如果你收到"Permission denied"的报错，你的`/usr/local/bin`目录可能是不可写的，你需要用超级用户来进行安装。运行`sudo -i`，然后执行下面的两个命令，然后`exit`.

下面这个例子：
```bash
 $ curl -L "https://github.com/docker/compose/releases/download/1.9.0/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
```
如果你对使用`curl`安装有疑问，你可以看看[备用的安装选项](https://docs.docker.com/compose/install/#alternative-install-options)
5.赋权
```bash
chmod +x /usr/local/bin/docker-compose
```
6.(可选)，为`bash`和`zsh`安装[命令补全](https://docs.docker.com/compose/completion/)
7.测试
```bash
$ docker-compose --version
 docker-compose version: 1.9.0
```

# 备用安装选项

## pip安装

这种方式是将compose当作一个python应用来从pip源中安装,建议使用[virtualenv](https://virtualenv.pypa.io/en/latest/)避免python冲突
```bash
$ pip install docker-compose
```

## 直接作为容器运行

既然docker-compose是一个python应用，那么当然可以作为一个容器来执行它

```bash
$ curl -L https://github.com/docker/compose/releases/download/1.9.0/run.sh > /usr/local/bin/docker-compose
$ chmod +x /usr/local/bin/docker-compose
```

# 主构建

如果你对预发布版本有兴趣，你可以从https://dl.bintray.com/docker-compose/master/ 下载二进制包。预发布版本有很多新的功能，但是不稳定。

# 升级

如果要把compose从1.2或更早的版本升级，你需要在升级compose后删除或迁移现有的容器，因为从1.3开始，compose使用docker labels来跟踪容器，因此需要使用添加的标签来重新创建它们。

如果compose检测到没有label的容器，那么它将拒绝运行，因此最终你不会有两套的容器。如果你要继续使用现有的容器(例如，因为他们有要保留的数据volume)，可以使用compose 1.5.x来迁移它们：
```bash
docker-compose migrate-to-labels
```

或者，如果你不想保留它们，你可以删除，compose只会创建新的
```bash
docker rm -f -v myapp_web_1 myapp_db_1 ..
```

# 卸载

卸载使用`curl`安装的compose
```bash
rm /usr/local/bin/docker-compose
```

卸载使用`pip`安装的compose
```bash
pip uninstall docker-compose
```
> 如果你收到"Permission denied"的报错，说明你没有权限删除`docker-compose`，你可以使用sudo来强制删除，或者使用超级管理员用户来删除。
