---
layout: post
title: jekyll 搭建静态博客之https番外篇
categories: [jekyll,travis]
description: 记录使用jekyll搭建博客的过程
keywords: jekyll,travis
---

> 上一篇HTTPS是使用cloudflare进行加密的，所有的请求都交给cloudflare来进行转发，就会出现域名指向的IP不是主机的问题，虽然安全些，但是总感觉怪怪的，所以这一篇，直接通过 nginx+docker+let's encrypt 搭建HTTPS认证

<!--more-->
### 前提

首先记得将 cloudflare 上有关你的站点的https配置删掉，如果没有做过，则忽略，同时记得去你的 DNS 控制器上将你的域名解析改为默认

### 搭建过程

搭建过程比较简单

- 准备好你前端的 nginx 服务器
- 准备好你后端的 docker 容器
- clone let's encrypt 到服务器
- 生成证书
- 配置你的 nginx

1.准备 nginx

```bash
yum update

yum install nginx
```

2.clone let's encrypt 到服务器

```bash
git clone https://github.com/certbot/certbot.git
```

3.准备 docker 容器

一条命令即可

```bash
docker run -d --name chinakevinguo_jekyll_kevinguo_me --restart=always -p 127.0.0.1:1080:80 chinakevinguo_jekyll_kevinguo_me
```

4.生成证书

**注意：如果你开了防火墙，记得一定要将防火墙关闭，否则生成证书的时候可能会报错**

```bash
systemctl stop firewalld

# 将这里的域名，换成你自己的域名
cd certbot
./certbot-auto certonly --nginx -d kevinguo.me -d www.kevinguo.me
```

成功后，内容如下

```bash
IMPORTANT NOTES:
 - Congratulations! Your certificate and chain have been saved at
   /etc/letsencrypt/live/kevinguo.me/fullchain.pem. Your cert will
   expire on 2017-10-11. To obtain a new or tweaked version of this
   certificate in the future, simply run certbot-auto again. To
   non-interactively renew *all* of your certificates, run
   certbot-auto renew
 - If you like Certbot, please consider supporting our work by:

   Donating to ISRG / Lets Encrypt:   https://letsencrypt.org/donate
   Donating to EFF:                    https://eff.org/donate-le
```

然后生成`dhparam`证书，可能会花费一段时间

```bash
openssl dhparam -out /etc/letsencrypt/live/kevinguo.me/dhparam.pem 2048
```

5.配置你的 nginx

nginx 的配置尽量模块化，这里通过 nginx 作为代理，访问后端的 docker 容器

**nginx.conf** 内容如下
```bash
user nginx;
worker_processes auto;
pid /run/nginx.pid;

# Load dynamic modules. See /usr/share/nginx/README.dynamic.
include /usr/share/nginx/modules/*.conf;

events {
    use epoll;
    worker_connections 1024;
}

http {
    server_tokens off;
    log_format  main  '$remote_addr||$time_local||"$request"||'
                      '$status||$body_bytes_sent||"$http_referer"'
                      '||$http_x_forwarded_for||'
                      '||$upstream_status||$upstream_addr||$request_time||$upstream_response_time||' ;

    sendfile            on;
    tcp_nopush          on;
    tcp_nodelay         on;
    keepalive_timeout   65;
    types_hash_max_size 2048;

    include             /etc/nginx/mime.types;
    default_type        application/octet-stream;

    gzip on;
    gzip_disable "msie6";
    gzip_proxied any;
    gzip_min_length 1000;
    gzip_comp_level 6;
    gzip_types text/plain text/css application/json application/x-javascript text/xml application/xml application/xml+rss text/javascript;

    include /etc/nginx/conf.d/*.conf;
    }

```

**对应站点.conf** 内容如下
```bash
upstream chinakevinguo_jekyll_kevinguo_me {
    server 127.0.0.1:1080;
}
server {
    listen 80 default_server;
    listen [::]:80 default_server;
    server_name kevinguo.me www.kevinguo.me;

    # Redirect all HTTP requests to HTTPS with a 301 Moved Permanently response.
    return 301 https://$server_name$request_uri;
    #return 404;
}

server {
    listen 443 ssl http2 default_server;
    listen [::]:443 ssl http2 default_server;
    server_name kevinguo.me www.kevinguo.me;

    # certs sent to the client in SERVER HELLO are concatenated in ssl_certificate
    ssl_certificate /etc/letsencrypt/live/kevinguo.me/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/kevinguo.me/privkey.pem;
    ssl_dhparam /etc/letsencrypt/live/kevinguo.me/dhparam.pem;
    ssl_session_timeout 1d;
    ssl_session_cache shared:SSL:50m;
    ssl_session_tickets off;
    ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
    ssl_ciphers 'ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-AES128-SHA256:ECDHE-RSA-AES128-SHA256:ECDHE-ECDSA-AES128-SHA:ECDHE-RSA-AES256-SHA384:ECDHE-RSA-AES128-SHA:ECDHE-ECDSA-AES256-SHA384:ECDHE-ECDSA-AES256-SHA:ECDHE-RSA-AES256-SHA:DHE-RSA-AES128-SHA256:DHE-RSA-AES128-SHA:DHE-RSA-AES256-SHA256:DHE-RSA-AES256-SHA:ECDHE-ECDSA-DES-CBC3-SHA:ECDHE-RSA-DES-CBC3-SHA:EDH-RSA-DES-CBC3-SHA:AES128-GCM-SHA256:AES256-GCM-SHA384:AES128-SHA256:AES256-SHA256:AES128-SHA:AES256-SHA:DES-CBC3-SHA:!DSS';
    ssl_prefer_server_ciphers on;
    ssl_stapling on;
    ssl_stapling_verify on;
    # HSTS (ngx_http_headers_module is required) (15768000 seconds = 6 months)
    add_header Strict-Transport-Security max-age=15768000;


    add_header X-Frame-Options DENY;
    add_header  X-Content-Type-Options  nosniff;
    add_header X-XSS-Protection "1";
    access_log /var/log/nginx/chinakevinguo_jekyll_kevinguo_me_access.log ;
    error_log /var/log/nginx/chinakevinguo_jekyll_kevinguo_me_error.log ;

    location / {
       root html;
       index index.html;
       proxy_pass         http://chinakevinguo_jekyll_kevinguo_me;
       proxy_set_header   Host             $host:443;
       proxy_set_header   X-Real-IP        $remote_addr;
       proxy_set_header   X-Forwarded-For  $proxy_add_x_forwarded_for;
       proxy_connect_timeout   10s;
       proxy_send_timeout      100s;
       proxy_read_timeout      300s;
       proxy_next_upstream error timeout http_404;
    }
}
```

最后，来访问试试看呢
