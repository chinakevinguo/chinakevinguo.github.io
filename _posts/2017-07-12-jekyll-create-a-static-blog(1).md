---
layout: post
title: jekyll 搭建静态博客(1)
categories: [jekyll,travis]
description: 记录使用jekyll搭建博客的过程
keywords: jekyll,travis
---

> 这段时间将原来在hexo上的博客迁到了jekyll上；采用Jekyll生成静态站点，Travis CI自动化部署，记录下来，以免以后忘记了

<!--more-->

### 整体思路

我们都知道通过jekyll搭建博客最终都是将通过`jekyll build`生成的`_site`下的静态文件发布出去，那么我们是不是可以直接采用 `nginx` + `静态文件`的方式来发布呢，当然是可以的

#### 准备两个站点仓库

- 主站点Github (保存静态文件)
- 镜像站点Github (触发 travis ，生成静态文件)

我们只需要将 Travis CI生成的静态文件推送到 主站点Github，博客通过 docker 化部署，采用 `nginx` + `静态文件` 方式；每次容器启动后都要从主站点Github pull 最新的静态文件，流程如下

- 本地提交博客 Markdown 文件到镜像站点Github
- Github 触发 Travis CI 执行jekyll 编译
- Travis CI 将编译后的静态文件push到主站点Github
- Travis CI 通知服务器重启容器
- 容器重启拉去最新静态文件完成更新


[Travis](https://travis-ci.org/) 是啥？ 就是个类似jenkins的东西. [Jenkins](https://jenkins.io/) 是啥？ 就是个类似Travis的东西.

#### 构建所需docker镜像

既然博客是通过 docker 化部署，采用`nginx` + `静态文件` 的方式发布，那么我们第一步就是要构建我们博客所需的镜像，`Dockerfile` 内容如下

```bash
FROM nginx:1.11.10-alpine

MAINTAINER KevinGuo "chinakevinguo@live.com"

ENV TZ 'Asia/Shanghai'

RUN apk upgrade --no-cache \
    && apk add --no-cache bash git \
    && rm -rf /usr/share/nginx/html \
    && git clone https://github.com/chinakevinguo/kevinguo.me.git /usr/share/nginx/html \
    && ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime \
    && echo "Asia/Shanghai" > /etc/timezone \
    && rm -rf /var/cache/apk/*

ADD entrypoint.sh /entrypoint.sh

WORKDIR /usr/share/nginx/html

CMD ["/entrypoint.sh"]
```

容器每次重启的时候都会去主站点Github上拉取最新的静态文件，并且启动 nginx，`entrypoint.sh`内容如下

```bash
#!/bin/bash

git pull
nginx -g "daemon off;"
```

最后执行`docker build`，生成我们所需的镜像

```bash
docker build -t chinakevinguo_jekyll_kevinguo_me .
```

博客所需镜像制作完成后，启动容器

```bash
docker run -d --name chinakevinguo_jekyll_kevinguo_me --restart=always -p 80:80 -p 443:443 chinakevinguo_jekyll_kevinguo_me
```
下一步就是 `静态文件` 了

#### 静态文件的自动更新

通过上面 `Dockerfile` 文件中的内容，你会发现，我是将 `kevinguo.me.git`下的内容clone到 `/usr/share/nginx/html`，也就是说，我实际上是发布的 `kevinguo.me.git` 下的静态文件，那么 `kevinguo.me.git` 下面的内容又是怎么来的呢

实际上这些文件是 Travis CI 基于镜像站点(chinakevinguo.github.io) 完成 build 后在 `_site`目录下生成的镜像文件

##### Travis 配置

1.使用 `Github` 账号登录 Travis，右上方按钮点击同步项目，下方打开需要继承的项目，最后点击齿轮进入项目配置页面

![travis](/images/posts/travis.png)

2.打开 `Build only if .travis.yml is present`

![travis-yml](/images/posts/travis-yml.png)

3.Travis CI push静态文件到Github 通过 Github的token实现授权配置，准备 Github上的token

**注意： 这里的`token`，复制之后，最好自己保存好哟，因为只显示一次，如果丢失只能再次生成了**

![github-token.png](/images/posts/github-token.png)

4.配置Travis CI 部署所需环境变量，`$JEKYLL_GITHUB_TOKEN`

![travis-token](/images/posts/travis-token.png)

5.Travis CI 和服务器之间通过密钥认证，并且对密钥进行了加密，所以我们需要在服务器上进行一些加密操作，并将密钥传到Travis上

```bash
# clone 镜像站点
git clone git@github.com:chinakevinguo/chinakevinguo.github.io.git

# 在镜像站点(chinakevinguo.github.io)下，新建`.travis.yml`文件
touch .travis.yml

# 生成公钥和私钥
ssh-keygen

# 安装travis命令行工具，如无法使用gem指令需先安装ruby
gem install travis

# --auto自动登录github账户
travis login --auto

# 在.travis.yml同级目录下执行，此处的--add参数表示自动添加脚本到.travis.yml文件中
travis encrypt-file ~/.ssh/id_rsa --add
```

执行完成后会发现`travis`网站项目里的环境变量多了两个参数

![travis-key](/images/posts/travis-key.png)

并且在`.travis.yml`里的 `before_install`周期中多了下面2行

```bash
- openssl aes-256-cbc -K $encrypted_3870315c7a22_key -iv $encrypted_3870315c7a22_iv
  -in id_rsa.enc -out ~\/.ssh/id_rsa -d
```

默认生成的命令会在`/`前面带转义符`\`，我们不需要，手动删除即可

6.进一步修改 `.travis.yml` 文件，内容如下

```bash
language: ruby
rvm:
- 2.3.3
before_install:
- openssl aes-256-cbc -K $encrypted_3870315c7a22_key -iv $encrypted_3870315c7a22_iv -in id_rsa.enc -out ~/.ssh/id_rsa -d
- chmod 600 ~/.ssh/id_rsa
- echo -e "Host 主机IP地址\n\tStrictHostKeyChecking no\n" >> ~/.ssh/config
install:
  - gem install jekyll
  - gem install html-proofer
script:
- bundle install
- bundle exec jekyll build
after_success:
- git clone https://github.com/chinakevinguo/kevinguo.me.git
- cd kevinguo.me && rm -rf * && cp -r ../_site/* .
- git config user.name "chinakevinguo"
- git config user.email "chinakevinguo@live.com"
- git add --all .
- git commit -m "Travis CI Auto Builder"
- git push --force https://${JEKYLL_GITHUB_TOKEN}@github.com/chinakevinguo/kevinguo.me.git master
- ssh root@host.kevinguo.me "docker restart chinakevinguo_jekyll_kevinguo_me"
branches:
  only:
  - master
env:
  global:
  - NOKOGIRI_USE_SYSTEM_LIBRARIES=true
addons:
  ssh_known_hosts: host.kevinguo.me
```

7.修改镜像站点下的其他内容，比如`_config.yml`，将一些内容替换成你自己的,下一篇，我将会介绍 `disqus` 和 `cloudflare`

8.所有内容都修改好后，`push`到 Github上，会发现触发了 Travis CI，并且将生成的静态文件 `push` 到了主站点仓库 `kevinguo.me`，然后重启了容器

![travis-done](/images/posts/travis-done.png)

博客发布成功

![jekyll-blog-done](/images/posts/jekyll-blog-done.png)
