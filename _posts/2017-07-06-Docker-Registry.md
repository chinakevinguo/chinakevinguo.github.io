---
layout: post
title: Docker基础-docker registry的几种方式
categories: [docker]
description: docker registry的几种常见方式
keywords: docker
---
# Registry
`Registry`用来管理您的`IMAGE`镜像，常见的有:
`Private Registry`:私有仓库，自己搭建
`DockerHub`：由`Docker`维护
`Docker Trusted Registry`：采购
<!--more-->
## 今天我们主要介绍，如何搭建自己的`Private Registry`
### Why use it
官网列了三条，其实就一句话，能够自己管理自己的IMAGE

### Storage
默认是存储在本地，所有的数据都作为`Docker volume`持久保存在本地主机上，适用于开发和小型部署，还支持其他云存储方式，如S3,Microsoft Azure,OpenStack Swift和Aliyun OSS，如果使用其他的存储方式，用户需要自己编写[Storage API](https://docs.docker.com/registry/storage-drivers/)。
### Running a domain registry
搭建`Private Registry`不仅仅是为了自己用，还可能给公司其他的人用，那这个时候就需要使用`TLS`来确保Registry的安全，有点类似于Web服务器的`SSL`
#### 纯HTTP的Registry
###### 从官方拉取一个Registry
```bash
$ docker run -d -p 5000:5000 --restart=always --name registry \
    -v `pwd`/data:/var/lib/registry \
    registry:2
```
###### 修改`Registry server`以及所有要访问`Registry server`的`client端`，让其支持HTTP传输(默认是HTTPS传输)
```bash
##1.12版
#Create or modify /etc/docker/daemon.json
$ { "insecure-registries":["youripordomain:5000"] }
#Restart docker daemon
$ systemctl restart docker
```
```bash
##1.10版
#Modify /etc/default/docker or /etc/sysconfig/docker,add or edit
$ INSECURE_REGISTRY='--insecure-registry youripordomain:5000'
#Restart docker daemon
$ systemctl restart docker
```
###### pull or push IMAGE from your Private Registry
```bash
$ docker pull ubuntu
$ docker tag ubuntu oo3p.com:5000/ubuntu
$ docker push oo3p.com:5000/ubuntu
$ docker pull oo3p.com:5000/ubuntu
```


#### 自签名证书的Registry
###### Generate a certificate
```bash
$ mkdir -p certs
$ openssl req \
    -newkey rsa:2048 -nodes -keyout certs/domain.key
    -x509 -days 365 -out certs/domain.crt
```
###### 将生成的domain.crt文件的内容，放入docker客户端系统的`ca-bundle`文件中，使操作系统信任我们的自签名证书，或者将domain.crt复制成docker客户端的/etc/docker/certs.d/www.oo3p.com:5000/ca.crt，两种方式皆可
```bash
$ cat ~/certs/domain.crt >>/etc/pki/tls/certs/ca-bundle.crt
$ systemctl restart docker
```
```bash
$ cp ~/certs/domain.crt /etc/docker/certs.d/www.oo3p.com:5000/ca.crt
$ systemctl restart docker
```

###### Start your Registry with TLS enabled
```bash
$ docker run -d -p 5000:5000 --restart=always --name registry \
    -v `pwd` /certs:/certs \
    -e REGISTRY_HTTP_TLS_CERTIFICATE=/certs/domain.crt \
    -e REGISTRY_HTTP_TLS_KEY=/certs/domain.key \
    registry:2
```
###### Pull or Push IMAGE from your Private Registry on another docker client
```bash
$ docker pull ubuntu
$ docker tag ubuntu oo3p.com:5000/ubuntu
$ docker push oo3p.com:5000/ubuntu
$ docker pull oo3p.com:5000/ubuntu
```

#### 从Lets Encrypt获取证书的Registry
从认证`CA`处获取签名证书，大多数需要付出一定的费用，这里可以使用`Lets Encrypt`(被大多数浏览器信任)提供的免费证书，和自签名证书不一样的是，我们不需要将domain.crt文件再复制到我们的docker客户端中了
**获取证书所需条件**
* Registry server需要有公网IP地址
* Registry server的80，443端口未被占用

###### Get letsencrypt
```bash
$ git clone https://github.com/certbot/certbot.git
$ cd certbot
$ ./certbot-auto certonly --standalone --email chinakevinguo@live.com -d oo3p.com -d www.oo3p.com

#看到这就表示已经获取证书成功，时间是90天，我们需要在到期之前手动续约，然后我们在`/etc/letsencrypt/live/域名`目录中可以看到4个文件(cert.pem chain.pem fullchain.pem privkey.pem)
IMPORTANT NOTES:
 - Congratulations! Your certificate and chain have been saved at
   /etc/letsencrypt/live/oo3p.com/fullchain.pem. Your cert will expire
   on 2017-02-23. To obtain a new or tweaked version of this
   certificate in the future, simply run certbot-auto again. To
   non-interactively renew *all* of your certificates, run
   "certbot-auto renew"
 - If you like Certbot, please consider supporting our work by:

   Donating to ISRG / Let's Encrypt:   https://letsencrypt.org/donate
   Donating to EFF:                    https://eff.org/donate-le
```

###### 查看证书，以及更新证书
```bash
#查看证书
$ ls /etc/letsencrypt/live/oo3p.com/
    cert.pem  chain.pem  fullchain.pem  privkey.pem

#更新证书
$ ./certbot-auto renew --dry-run
```

###### 将从letsencrypt获取到的证书，复制到Registry server上的/certs目录下，方便挂载到Registry 容器里
```bash
cp /etc/letsencrypt/live/oo3p.com/fullchain.pem ~/certs/domain.crt
cp /etc/letsencrypt/live/oo3p.com/privkey.pem   ~/certs/domain.key
```

###### Start your Registry with TLS enabled
```bash
$ docker run -d -p 5000:5000 --restart=always --name registry \
    -v `pwd` /certs:/certs \
    -e REGISTRY_HTTP_TLS_CERTIFICATE=/certs/domain.crt \
    -e REGISTRY_HTTP_TLS_KEY=/certs/domain.key \
    registry:2
```
###### Pull or Push IMAGE from your Private Registry on another docker client
```bash
$ docker pull ubuntu
$ docker tag ubuntu oo3p.com:5000/ubuntu
$ docker push oo3p.com:5000/ubuntu
$ docker pull oo3p.com:5000/ubuntu
```

#### Load Balancing 注意事项以及访问控制

###### 在做Registry集群的时候，下面这几个必须保持相同：
* Storage Driver
* HTTP Secret
* Redis Cache(if configured)

###### 本机基本访问认证
**在这之前，你需要先配置了TLS认证**
```bash
$ mkdir auth -p
# 创建密码文件，替换用户名：testuser，密码：testpassword
$ docker run --entrypoint htpasswd registry:2 -Bbn testuser testpassword > auth/htpasswd
# 停止registry，然后用下面的代码再次启动
$ docker run -d -p 5000:5000 --restart=always --name registry \
  -v /data/registry:/var/lib/registry \
  -v ~/auth:/auth \
  -e "REGISTRY_AUTH=htpasswd" \
  -e "REGISTRY_AUTH_HTPASSWD_REALM=Registry Realm" \
  -e REGISTRY_AUTH_HTPASSWD_PATH=/auth/htpasswd \
  -v ~/certs:/certs \
  -e REGISTRY_HTTP_TLS_CERTIFICATE=/certs/domain.crt \
  -e REGISTRY_HTTP_TLS_KEY=/certs/domain.key \
  registry:2
# 然后登录
$ docker login www.oo3p.com:5000
```
###### 通过代理进行认证
* [using Apache as an authenticating proxy](https://docs.docker.com/registry/recipes/apache/)
* [using Nginx as an authenticating proxy](https://docs.docker.com/registry/recipes/nginx/)
* [mirror the Docker Hub](https://docs.docker.com/registry/recipes/mirror/)

###### 通过委派第三方认证
将用户重定向到指定的可信任的服务器上进行认证，这比较耗费成本
* [background information here](https://docs.docker.com/registry/spec/auth/token/)
* [configuration information here](https://docs.docker.com/registry/configuration/#auth)

#### 使用Dockerfile构建自己的Registry

###### 从docker的Github上拉取有关docker-registry的代码
```bash
$ git clone https://github.com/docker/distribution-library-image.git
```

###### 修改distribution-library-image下的Dockerfile文件
```bash
FROM alpine:3.4

RUN set -ex \
    && apk add --no-cache ca-certificates apache2-utils

COPY ./registry/registry /bin/registry
COPY ./registry/config-example.yml /etc/docker/registry/config.yml
# 这里我们自己添加，后面将在certs目录下生成证书
COPY ./registry/certs /etc/ssl/docker

VOLUME ["/var/lib/registry"]
EXPOSE 5000

COPY docker-entrypoint.sh /entrypoint.sh
ENTRYPOINT ["/entrypoint.sh"]

CMD ["/etc/docker/registry/config.yml"]
```

###### 生成自签名证书
```bash
$ mkdir -p ~/distribution-library-image/registry/certs
$ openssl req -newkey rsa:2048 -nodes -keyout ~/distribution-library-image/registry/certs/docker_registry.key -x509 -days 365 -out ~/distribution-library-image/registry/certs/docker_registry.crt
```

###### 修改distribution-library-image/registry 下的config-example.yml
```bash
$ cp -a config-example.yml config-example.yml.bak
$ vi config-example.yml
version: 0.1
log:
  accesslog:
    disabled: true
  level: debug
  formatter: text
  fields:
    service: registry
# 该处设置hook，触发邮件发送
  # hooks:
  #   - type: mail
  #     disabled: true
  #     levels:
  #       - panic
  #     options:
  #       smtp:
  #         addr: mail.example.com:25
  #         username: mailuser
  #         password: password
  #         insecure: true
  #       from: sender@example.com
  #       to:
  #         - errors@example.com
# 该处指定存储方式和位置，我是直接使用的filesystem
storage:
  filesystem:
    rootdirectory: /var/lib/registry
    maxthreads: 100
  delete:
    enabled: false
  redirect:
    disable: false
  cache:
    blobdescriptor: inmemory
# 该处指定你的清楚策略
  maintenance:
  # 表示上传168h后删除，没隔1天清除一次
    uploadpurging:
      enabled: true
      age: 168h
      interval: 24h
      dryrun: false
    readonly:
      enabled: false
http:
  addr: :5000
#  host: https://registry.quarkfinance.com:5000
# 该处指定你的证书所在的位置
tls:
  certificate: /etc/ssl/docker/docker_registry.crt
  key: /etc/ssl/docker/docker_registry.key
headers:
  X-Content-Type-Options: [nosniff]
health:
storagedriver:
  enabled: true
  interval: 10s
  threshold: 3
# 该处指定你的代理，如果你有的话
#proxy:
#  remoteurl: https://registry-1.docker.io
#  username: [username]
#  password: [password]
```
更多内容参考[configuring a registry](https://docs.docker.com/registry/configuration/)

###### 开始构建(该命令在Dockerfile文件所在的目录执行)
```bash
$  docker build -t registry.quarkfinance.com:5000/docker_registry:latest .
```

###### 启动docker registry(我这里挂载了hosts目录，是因为我的registry是通过域名访问的，而我的域名实际上是不存在的，所以，我只能在/etc/hosts文件中添加了隐射，后期如果有实际存在的域名解析，就不需要挂载了)
```bash
$ docker run -d -p 5000:5000 --name docker_registry --privileged -v /etc/hosts:/etc/hosts --restart always registry.quarkfinance.com:5000/docker_registry
```

###### 如何使用

 在所有要访问docker registry的主机上执行如下操作
```bash
$ mkdir -p /etc/docker/certs.d/registry.quarkfinance.com:5000
```

 将registry主机上生成的证书`docker_registry.crt`复制到所有要访问docker registry的主机上
```bash
$ ip=(192.168.1.1 192.168.1.2 192.168.1.3)
$ for i in ${ip[*]};
    \ do
    \ scp ~/distribution-library-image/registry/certs/docker_registry.crt
    \ $i:/etc/docker/certs.d/registry.quarkfinance.com:5000/
    \ done
```

 试着push
```bash
[root@k8s-master01 ~]# docker push registry.quarkfinance.com:5000/nginx:1.11.4-alpine
The push refers to a repository [registry.quarkfinance.com:5000/nginx]
e83e87cb73c3: Pushed
740e5e49ea89: Pushed
092e6cb28bdb: Pushed
9007f5987db3: Pushed
1.11.4-alpine: digest: sha256:36b95728c0d9b6668bb8e4aa31476de27aeedbeba07dcddb6821b76fb72830b2 size: 1154
```

发现push成功了,至此你的私有registry就算是成功了，下面我们来使用nginx来做认证

#### nginx+registry认证登录

###### 从docker的Github上拉取有关docker-registry的代码
```bash
$ git clone https://github.com/docker/distribution-library-image.git
```

###### 生成自签名证书
```bash
$ mkdir -p ~/distribution-library-image/registry/certs
$ openssl req -newkey rsa:2048 -nodes -keyout ~/distribution-library-image/registry/certs/docker_registry.key -x509 -days 365 -out ~/distribution-library-image/registry/certs/docker_registry.crt
```

###### 修改distribution-library-image/registry 下的config-example.yml
```bash
$ cp -a config-example.yml config-example.yml.bak
$ vi config-example.yml
version: 0.1
log:
  accesslog:
    disabled: true
  level: info
  formatter: text
  fields:
    service: registry
# 该处设置hook，触发邮件发送
  # hooks:
  #   - type: mail
  #     disabled: true
  #     levels:
  #       - panic
  #     options:
  #       smtp:
  #         addr: mail.example.com:25
  #         username: mailuser
  #         password: password
  #         insecure: true
  #       from: sender@example.com
  #       to:
  #         - errors@example.com
# 该处指定存储方式和位置，我是直接使用的filesystem
storage:
  filesystem:
    rootdirectory: /var/lib/registry
    maxthreads: 100
  delete:
    enabled: false
  redirect:
    disable: false
  cache:
    blobdescriptor: inmemory
# 该处指定你的清楚策略
  maintenance:
  # 表示上传168h后删除，没隔1天清除一次
    uploadpurging:
      enabled: true
      age: 168h
      interval: 24h
      dryrun: false
    readonly:
      enabled: false
http:
  addr: :5000
#  host: https://registry.quarkfinance.com:5000
# 该处指定你的证书所在的位置，注释掉，因为我们这里使用nginx的认证，如果指定两处，会出问题
#tls:
#  certificate: /etc/ssl/docker/docker_registry.crt
#  key: /etc/ssl/docker/docker_registry.key
headers:
  X-Content-Type-Options: [nosniff]
health:
storagedriver:
  enabled: true
  interval: 10s
  threshold: 3
# 该处指定你的代理，如果你有的话
#proxy:
#  remoteurl: https://registry-1.docker.io
#  username: [username]
#  password: [password]
```
更多内容参考[configuring a registry](https://docs.docker.com/registry/configuration/)

###### 开始构建(该命令在Dockerfile文件所在的目录执行)
```bash
$  docker build -t registry.quarkfinance.com/docker_registry:latest .
```

###### 启动docker registry(我这里挂载了hosts目录，是因为我的registry是通过域名访问的，而我的域名实际上是不存在的，所以，我只能在/etc/hosts文件中添加了隐射，后期如果有实际存在的域名解析，就不需要挂载了),同时需要指定监听的ip，不让其它主机访问registry的5000端口，其它主机通过nginx代理访问
```bash
$ docker run -d -p 127.0.0.1:5000:5000 --name docker_registry --privileged -v /etc/hosts:/etc/hosts --restart always registry.quarkfinance.com:5000/docker_registry
```
###### 配置nginx

 安装nginx
```bash
# 配置nginx的yum repo
vi /etc/yum.repos.d/nginx.repo
[nginx]
name=nginx repo
baseurl=http://nginx.org/packages/centos/$releasever/$basearch/
gpgcheck=0
enabled=1

# 安装nginx
$ yum install nginx -y         
```

 配置nginx的配置文件/etc/nginx/conf.d/(不同版本的nginx配置文件有所不同)
```bash
[root@k8s-registry conf.d]# ll
total 12
-rw-r--r-- 1 root root 1097 Apr 12 23:21 default.conf
-rw-r--r-- 1 root root  933 Apr 13 18:24 registry.conf
-rw-r--r-- 1 root root   46 Apr 13 15:11 registry-upstream.conf
```

```bash
# registry.conf

server {
    listen       443;
    server_name  registry.quarkfinance.com;
    # 指定证书的位置，这里我们使用的就是上面生成的证书
    ssl          on;
    ssl_certificate /etc/nginx/ssl/docker_registry.crt;
    ssl_certificate_key /etc/nginx/ssl/docker_registry.key;
    client_max_body_size 0;

   # charset koi8-r;
   # access_log  /var/log/nginx/log/host.access.log  main;

    location / {
        auth_basic "Registry realm";
        # 指定认证文件
        auth_basic_user_file /etc/nginx/conf.d/.htpasswd;
        add_header 'Docker-Distribution-Api-Version' 'registry/2.0' always;
        proxy_pass                          http://registry;
        proxy_set_header  Host              $http_host;   # required for docker client's sake
        proxy_set_header  X-Real-IP         $remote_addr; # pass on real client's IP
        proxy_set_header  X-Forwarded-For   $proxy_add_x_forwarded_for;
        proxy_set_header  X-Forwarded-Proto $scheme;
        proxy_read_timeout                  900;
    }

}

```

```bash
# registry-upstream.conf

upstream registry {
        server 127.0.0.1:5000;
}                             
```

 生成认证文件.htpasswd

```bash
$ yum install httpd-tool -y

$ htpasswd -cb /etc/nginx/conf.d/.htpasswd ${username} ${password}
```

 重启nginx，就算是完成了

```bash
$ systemctl restart nginx
```

###### 如何使用

 在所有要访问docker registry的主机上执行如下操作，目的是为了让其他主机信任registry主机签发的证书
```bash
$ mkdir -p /etc/docker/certs.d/registry.quarkfinance.com
```

 将registry主机上生成的证书`docker_registry.crt`复制到所有要访问docker registry的主机上
```bash
$ ip=(192.168.1.1 192.168.1.2 192.168.1.3)
$ for i in ${ip[*]};
    \ do
    \ scp ~/distribution-library-image/registry/certs/docker_registry.crt
    \ $i:/etc/docker/certs.d/registry.quarkfinance.com/
    \ done
```

 试着push，提示没有认证
```bash
[root@k8s-master02 registry.quarkfinance.com]# docker push registry.quarkfinance.com/calico/node:v1.1.0-rc8
The push refers to a repository [registry.quarkfinance.com/calico/node]

a48093c264c7: Preparing
ca135d205d41: Preparing
cb8d3fea875c: Preparing
24a574387e35: Preparing
no basic auth credentials
```

 我们先登录
```bash
[root@k8s-master02 registry.quarkfinance.com]# docker login registry.quarkfinance.com
Username: admin
Password:
Login Succeeded

```

 再push
```bash
[root@k8s-master02 registry.quarkfinance.com]# docker push registry.quarkfinance.com/calico/node:v1.1.0-rc8
The push refers to a repository [registry.quarkfinance.com/calico/node]
a48093c264c7: Pushed
ca135d205d41: Pushed
cb8d3fea875c: Pushed
24a574387e35: Pushed
9f8566ee5135: Pushed
v1.1.0-rc8: digest: sha256:234b00b8712c80d0e8d0170c6cc65c005c0057484fe2c463d439f37ddb1fd889 size: 1371
```
