---
layout: post
title: jekyll 搭建静态博客(2)
categories: [jekyll,travis]
description: 记录使用jekyll搭建博客的过程
keywords: jekyll,travis
---

> 接上一篇，上一篇我们的博客已经可以自动化部署了，但是我们仍然不满足，还想要有一个好的评论系统和一个安全的https连接

<!--more-->

### Disqus

disqus 是一家第三方社会化评论系统，主要为网站主提供评论托管服务

1.在[disqus官网](https://disqus.com/)注册一个账号

2.点击`Admin`，然后新建站点，在`Website Name`处输入你的站点名称，并且根据你的情况选择站点种类

![disqus-create](/images/posts/disqus-create.png)

3.在`Website URL`处输入你的站点 url，保存即可

![disqus-conf](/images/posts/disqus-conf.png)

4.修改你的`_config.yml`文件中的 `comments_provider` 和 `username`

**重要：** 这里的`username`，实际上是 disqus 中的shortname

```bash
comments_provider: disqus
disqus:
    username: kevinguo
```

5.修改完成后，push 到镜像站点，触发 Travis CI 重新发布博客，最终成功加载 disqus

![disqus-done](/images/posts/disqus-done.png)

如果无法加载 disqus ，可能是因为被墙了，FQ 出去试试

### Cloudflare

cloudflare 主要是为客户提供网站安全管理，性能优化等，比如 `HTTPS`,`CDN`

1.在[godaddy](https://sg.godaddy.com/zh/)上去申请一个域名吧，一年也就 5$

2.在[cloudflare官网](https://www.cloudflare.com)注册一个账号

3.点击 `Add Site`，添加一个站点，然后 `Begin Scan`,大概需要60s

![cloudflare-add](/images/posts/cloudflare-add.png)

4.扫描完成后，会看到 DNS 记录，自行添加(其中彩色的云朵表示开启SSL，否则就只是DNS)

**注意** 如果你的A记录启用的SSL，那么所有关于这个A记录的请求都会转发给 Cloudflare，然后通过 Cloudflare再转发到你的服务器，所以这个时候，你通过 `nslookup kevinguo.me` 的时候解析出来的地址，并不是你的服务器地址，而是 Cloudflare的地址;如果你有其他服务(诸如VPS,FTP等)使用的是这个地址的话，最好是再添加一条不走SSL的A记录

![cloudflare-dns](/images/posts/cloudflare-dns.png)

5.完成上面的步骤后，到你的域名控制面板修改DNS服务

![godaddy-dns](/images/posts/godaddy-dns.png)

6.修改完成后，在 Cloudflare点击继续，大概5~30分钟后 `Overview` 状态会变成 `Status: Active`

![cloudflare-status](/images/posts/cloudflare-status.png)

7.点击 `Crypto`来设置SSL 级别

![cloudflare-ssl](/images/posts/cloudflare-ssl.png)

8.点击 `Page Rules`来设置域名重定向

- 将顶级域名都重定向到 `https://www.kevinguo.me`

![cloudflare-forwarding](/images/posts/cloudflare-forwarding.png)

- 添加自动使用 HTTPS，所有访问`http://www.kevinguo.me`的请求都使用HTTPS

![cloudflare-always-https](/images/posts/cloudflare-always-https.png)

9.最后，访问你的blog试试呢
