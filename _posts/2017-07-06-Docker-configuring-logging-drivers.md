---
layout: post
title: Docker基础-docker日志驱动配置
categories: [docker]
description: docker日志驱动配置
keywords: docker
---
Docker 包含多个日志记录机制，来帮助你从运行的容器或服务获取日志信息。这些机制成为日志驱动。
每一个docker daemon都有一个默认的日志驱动，每个容器都默认都是使用这个日志驱动，除非你自定义了其他的日志驱动。

# 为docker daemon配置 默认的日志驱动
要将docker daemon配置为默认的日志驱动程序，请使用`--log-driver=<VALUE>`选项。如果驱动程序有可配置的选项，则可以使用一个或多个`--log-opt<NAME>=<VALUE>`来进行设置。
如果你没有指定日志驱动，那么默认是`json-file`。因此诸如`docker inspect`等命令输出的内容都是json格式的。

下面我们看看当前默认的日志驱动是啥
```bash
$ docker info |grep 'Logging Driver'

Logging Driver: json-file
```
<!--more-->
# 为容器配置日志驱动
当启动容器时，我们可以为容器配置和docker daemon不一样的日志驱动。如果驱动程序有可配置的选项，则可以使用一个或多个`--log-opt<NAME>=<VALUE>`来进行设置。即使容器使用默认的日志驱动，也可以使用不同的配置选项。

看看当前容器的日志驱动是啥
```bash
$ docker inspect -f {{.HostConfig.LogConfig.Type}} webtest1
json-file
```

# 所支持的日志驱动如下

| Driver | Description |
| :----- | :---------- |
| none | 无日志 |
| json-file | 将日志写入json-file，默认值 |
| syslog | 将日志写入syslog，syslog必须在机器上启动 |
| journald | 将日志写入journald,journald必须在机器上启动 |
| gelf | 将日志写入GELF端点，如Graylog或Logstash |
| fluentd | 将日志吸入fluentd，fluentd必须在机器上启动 |
| awslogs | 将日志写入亚马逊Cloudwatch |
| splunk | 使用HTTP事件收集器将日志写入splunk |
| etwlogs | 将日志消息作为windows时间跟踪。仅在windows平台可用 |
| gcplogs | 将日志写入Google云平台 |
| nats | 将日志发布到NATS服务器 |

# 日志驱动的限制

* `docker logs`命令不能用于`json-file`和`journald`之外的日志驱动

# Examples

## 使用label标签或者环境变量来配置日志驱动
如果你的容器在Dockerfile里或者在运行时，指定了标签或者环境变量。一些日志驱动就可以使用这些标签或环境变量来控制日志行为。如果标签和环境变量冲突，环境变量优先。

在启动Docker daemon时指定属性和选项。例如，手动启动daemon，指定使用json-file驱动，并且设定一个标签和两个环境变量，如下：
```bash
$ dockerd \
          --log-driver=json-file \
          --log-opt labels=production_status \
          --log-opt env=os,customer
```

下一步，运行一个容器，并指定labels和env的值，如下：
```bash
$ docker run -tid --label production_status=testing -e os=ubuntu alpine sh
```

最后，我们来查看记录到本地json-file中的日志内容
```bash
$ cat /var/lib/docker/containers/02bb794219c31704629bd0c84d39d56228b8163bc8d4925002c5164b678c521a/02bb794219c31704629bd0c84d39d56228b8163bc8d4925002c5164b678c521a-json.log
"os":"ubuntu","production_status":"test123"
```
如果日志驱动支持，则会想日志记录输出添加的附加字段，一下输出用于json文件：
```bash
"attrs":{"production_status":"testing","os":"ubuntu"}
```

## none
none 驱动，禁用了docker daemon的日志记录或在运行时禁用了单个容器的日志记录

```bash
$ docker run -ti --log-driver none alpine sh
```

## json-file
json-file 是默认的日志驱动，而且返回的是json格式的输出

json-file日志驱动支持如下选项：

| Options | Description | Example value |
| :-- | :-- | :-- |
| max-size | 每个日志大小 | --log-opt max-size=10m |
| max-file | 可以存在多少个日志文件,仅在同时设置了max-size时有效 | --log-opt max-file=3 |
| labels | 在启动docker daemon时使用，为日志内容添加一个有关标签的，逗号分隔的列表 | --log-opt labels=production_status,geo |
| env | 在启动docker daemon时使用，为日志提供和日志记录相关的环境变量 | --log-opt env=os,customer |

```bash
$ docker run -ti --log-opt max-size=10m --log-opt max-file=3 alpine sh
```

## syslog

下面是支持syslog日志驱动的选项：

| Option | Description | Example value |
| :--- | :--- | :--- |
| syslog-address | 外部syslog服务器地址，指定方式有[tcp/udp/tcp+tls]://host:port，unix://path 或 unixgram://path，默认的传输端口514 | --log-opt syslog-address=tcp+tls://192.168.1.3:514,--log-opt syslog-address=///tmp/syslog.sock |
| syslog-facility | 要使用的syslog设施。可以是任何有效的syslog工具的编号或名称。可参考[syslog 文档](https://tools.ietf.org/html/rfc5424#section-6.2.1) | --log-opt syslog-facility=daemon |
| syslog-tls-ca-cert | CA签发的信任证书的绝对路径。如果地址协议不是tcp+tls，则忽略 | --log-opt syslog-tls-ca-cert=/etc/ca-certificates/custom/ca.pem |
| syslog-tls-cert | TLS证书文件的绝对路径。如果地址协议不是tcp+tls,则忽略 | --log-opt syslog-tls-cert=/etc/ca-certificates/custom/cert.pem |
| syslog-tls-key | TLS key文件的绝对路径。如果地址协议不是tcp+tls，则忽略 | --log-opt syslog-tls-key=/etc/ca-certificates/custom/key.pem |
| syslog-tls-skip | 如果设置为true，则在连接到syslog守护程序时，跳过TLS验证。默认为false。如果地址协议不是tcp+tls，则忽略 | --log-opt syslog-tls-skip-verify=true |
| tag | 附加到syslog日志中APP-NAME的字符串。默认情况下，docker使用容器ID的前12个字符来标记日志信息。 | --log-opt tag=mailer |
| syslog-format | 要使用的syslog消息格式。如果没指定，则使用本地UNIX syslog格式，而不是用指定的主机名。比如：为RFC-3164兼容格式指定rfc3164，为RFC-5424兼容格式指定rfc5424 | --log-opt syslog-format=rfc5424micro |
| labels | 在启动docker daemon时使用，为日志内容添加一个有关标签的，逗号分隔的列表  | --log-opt labels=production_status,geo |
| env | 在启动docker daemon时使用，为日志提供和日志记录相关的环境变量 | --log-opt env=os,customer |

```bash
$ docker run \
         --log-driver=syslog \
         --log-opt syslog-address=tcp://192.168.0.42:123 \
         --log-opt syslog-facility=daemon \
         alpine ash
```


```bash
$ docker run \
         --log-driver=syslog \
         --log-opt syslog-address=tcp+tls://192.168.0.42:123 \
         --log-opt syslog-tls-ca-cert=syslog-tls-ca-cert=/etc/ca-certificates/custom/ca.pem \
         --log-opt syslog-tls-cert=syslog-tls-ca-cert=/etc/ca-certificates/custom/cert.pem \
         --log-opt syslog-tls-key=syslog-tls-ca-cert=/etc/ca-certificates/custom/key.pem \
         alpine ash
```

## journald

journald 日志驱动将容器ID存储在日志的CONTAINER_ID字段中。油光journald的详细信息，请参考[the journald logging driver](https://docs.docker.com/engine/admin/logging/journald/)

| Option | Description | Example vaule |
| :----- | :---------- | :------------ |
| tag | 用于在journald日志中设置CONTAINER_TAG的值的模板 | --log-opt tag=mailer |
| labels | 在启动docker daemon时使用，为日志内容添加一个有关标签的，逗号分隔的列表 | --log-opt labels=production_status,get |
| env | 在启动docker daemon时使用，为日志提供和日志记录相关的环境变量 | --log-opt env=os,customer |

```bash
$ docker run \
         --log-driver=journald \
         alpine ash
```

这里我就列几个常用的，还有很多，可以参考[docker 官网](https://docs.docker.com/engine/admin/logging/overview/)
