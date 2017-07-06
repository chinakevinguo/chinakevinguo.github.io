---
layout: post
title: saltstack之pillar
categories: [saltstack]
description: saltstack之pillar
keywords: saltstack
---
# **Pillar**
主要作用是存储和定义配置管理中需要的一些变量，比如数据，比如软件版本，用户名密码等信息

<!--more-->
###### 1.在master上开启pillar_roots

```yaml
pillar_roots:
  base:
    - /srv/pillar
```
<!--more-->
###### 2.在master的/srv/pillar目录下新建*top.sls*，*packages.sls*，*services.sls*

```yaml
[root@saltstack-node1 pillar]# cat top.sls
base:
  '*':
    - packages
    - services
```

```yaml
[root@saltstack-node1 pillar]# cat packages.sls
zabbix:
  package-name: zabbix
  version: 2.2.4
```

```yaml
[root@saltstack-node1 pillar]# cat services.sls
zabbix:
  port: 10050
  user: admin
```

###### 3.执行pillar.item zabbix查看指定的信息

```yaml
[root@saltstack-node1 pillar]# salt 'saltstack-node2*' pillar.item zabbix
saltstack-node2.example.com:
    ----------
    zabbix:
        ----------
        package-name:
            zabbix
        port:
            10050
        user:
            admin
        version:
            2.2.4

```
