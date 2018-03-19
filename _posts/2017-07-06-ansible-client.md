---
layout: post
title: ansible客户端配置
categories: [ansible, kargo]
description: ansible客户端配置
keywords: kargo,kubernetes,docker,ansible
---

为kargo kubernetes准备ansible客户端

<!--more-->

1.ansible-client 免密钥登录所有要部署的节点
```bash
$ ssh-keygen -t rsa -P ''
```
2.将ansible-client上的id_rsa.pub复制到其他所有节点
```bash
$ IP=(172.30.33.89 172.30.33.90 172.30.33.91 172.30.33.92 172.30.33.93 172.30.33.94)

$ for x in ${IP[*]}; do ssh-copy-id -i ~/.ssh/id_rsa.pub $x; done
```

2.ansible-client 上安装pip,python,setuptools,最好能先`yum update`一下
```bash
# setuptools安装
$ wget https://bootstrap.pypa.io/get-pip.py
$ python get-pip.py

# pip安装
$ wget https://pypi.python.org/packages/11/b6/abcb525026a4be042b486df43905d6893fb04f05aac21c32c638e939e447/pip-9.0.1.tar.gz#md5=35f01da33009719497f01a4ba69d63c9
$ tar zxvf pip-9.0.1.tar.gz
$ python setup.py install
```

3.ansible安装
```bash
$ yum install pycrypto python2-cryptography python-netaddr epel-release python-pip python34 python34-pip -y
$ pip install ansible
```
