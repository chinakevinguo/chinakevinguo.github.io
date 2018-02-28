---
layout: post
title: 手动搭建kubernetes HA集群(二)
categories: [kubernetes, docker,fabio,registrator,consul]
description: 手动搭建kubernetes HA集群
keywords: kubernetes,docker,fabio,registrator,consul
---

> 我们在第一章的时候，通过手动的方式搭建好了kubernetes集群，并且在上面跑了一些基础的服务，那么我们要如何将这些服务暴露出来呢，这一章重点介绍关于kubernetes的服务暴露

### 一、Kubernetes 服务暴露介绍

关于服务暴露，常见的有如下几种方式:

* LoadBlancer Service
* NodePort Service
* Ingress
* traefik

#### 1.1、LoadBlancer Service

LoandBlancer Service 是kubernetes深度结合云平台的一个组件；当使用LoandBlancer Service暴露服务时，实际上是通过向底层云平台申请创建一个负载均衡器来向外暴露服务；目前LoadBlancer Service支持的云平台已经相对完善，比如国外的GCE、DigitalOcean，国内的阿里云，私有云Openstack等等，由于LoadBlancer Service深度结合了云平台，所以只能在一些云平台使用。

#### 1.2、NodePort Service

NodePort Service 顾名思义，实质上就是通过在集群的每个node上暴露一个端口，然后将这个端口隐射到某个具体的service来实现，虽然每个node的端口有很多(0~65535)，但由于安全性和易用性(服务多了，端口记不住，容易混乱)，实际上使用的可能并不多

#### 1.3、Ingress

ingress 这东西在1.2后才出现的，大致原理就是通过一个ingress controller来实时感知service、pod的变化，然后结合ingress生成配置，更新内部的反代，刷新配置，达到服务暴露的目的

#### 1.4、traefik

traefik 笔者并没有使用过，大致意思是抛弃了ingress controller，因为traefik本身就能和kubernetes API交互，感知后端变化，再根据ingress生成规则，暴露服务。

### 二、fabio+consul+registrator 实现服务暴露

> 前面简单的介绍了几种kubernetes中的服务暴露方式，但是我这里一种都没有使用，为什么呢，**因为通过service或者Nodeport来实现服务发现都是使用的iptables来进行负载的，性能上总是有些损耗的**，所以这里我使用consul+registrator+fabio来实现kubernetes内部服务的暴露

组件介绍：

* fabio
* registrator
* consul

#### 2.1、fabio

fabio 是ebay团队用golang开发的一个快速、简单零配置就能够让consul部署的应用快速支持http(s)的负载均衡路由器，支持服务发现，自动生成路由
我们只需要在consul注册服务，提供一个健康检查，fabio就会将流量路由到这些服务上

#### 2.2、registrator

registrator(注册器)，能够实时的监听docker的event，动态的注册docker 容器服务到consul、etcd或zookeeper中

#### 2.3、consul

Consul 是一个支持多数据中心分布式高可用的服务发现和配置共享的服务软件,由 HashiCorp 公司用 Go 语言开发, 基于 Mozilla Public License 2.0 的协议进行开源. Consul 支持健康检查,并允许 HTTP 和 DNS 协议调用 API 存储键值对


#### 2.4、具体部署过程

具体架构图如下：

![fabio-consul-registrator](/images/posts/fabio-consul-registrator.png)

首先我们需要部署一套`consul server cluster`，具体部署过程，这里就不再演示了，请参考[Consul 集群搭建](https://github.com/kaizamm/consul/blob/master/consul%2Bdocker%2Bregistrator.md)

然后，我们所有的节点都需要包含在`calico-network`范围之内，calico网络部署请参考[第一章](https://kevinguo.me/2017/09/22/manual-deploy-kubernetes/),将所有fabio所在的节点配置为noscheduler

1.consul client 部署

> 我们在每个节点上跑一个consul-client，你可以以daemonset的方式部署，也可以直接以二进制的方式部署，用systemd管理起来，这里用二进制的方式

**/usr/lib/systemd/system/consul.service**

```bash
[Unit]
Description=Consul service discovery agent
Requires=network-online.target
After=network-online.target
[Service]
User=consul
Group=consul
EnvironmentFile=-/etc/default/consul
Environment=GOMAXPROCS=2
Restart=on-failure
ExecStartPre=[ -f "/opt/consul/run/consul.pid" ] && /usr/bin/rm -f /opt/consul/run/consul.pid
ExecStart=/usr/local/bin/consul agent $CONSUL_FLAGS
ExecReload=/bin/kill -HUP $MAINPID
KillSignal=SIGTERM
TimeoutStopSec=5
[Install]
WantedBy=multi-user.target
```

**/etc/default/consul**

```bash
CONSUL_FLAGS="-ui -data-dir=/opt/consul/data -config-dir=/opt/consul/conf -pid-file=/opt/consul/run/consul.pid -client=0.0.0.0 -bind=172.29.151.4 -node=consul-client04 -retry-join=172.30.33.39 -retry-join=172.30.33.40 -retry-join=172.30.33.41 -retry-interval=3s"
```

2.registrator 部署

> 我们在每个node 节点上跑一个registrator，同样可以以daemonset的方式部署，或者使用docker container的方式部署，用systemd管理，这里我们通过daemonset的方式部署

**registrator.yaml**

```yaml
apiVersion: extensions/v1beta1
kind: DaemonSet
metadata:
  creationTimestamp: null
  labels:
    run: registrator
  name: registrator
spec:
  selector:
    matchLabels:
      run: registrator
  template:
    metadata:
      creationTimestamp: null
      labels:
        run: registrator
    spec:
      volumes:
      - name: docker-sock
        hostPath:
          path: /var/run/docker.sock
      hostNetwork: true
      containers:
        - name: registrator
          volumeMounts:
          - mountPath: /tmp/docker.sock
            name: docker-sock
          image: ganeshkaila/registrator:v7
          command: ["/bin/sh"]
          args: ["-c", "registrator -useIpFromEnv=POD_IP -internal consul://localhost:8500"]
          imagePullPolicy: IfNotPresent
          env:
            - name: NODE_NAME
              valueFrom:
                fieldRef:
                  fieldPath: spec.nodeName
```

3.fabio 部署

> 我们在对应的节点上部署fabio，二进制文件部署，通过systemd管理

**fabio.service**

```bash
[Unit]
Description=fabio
Requires=network-online.target
After=network-online.target

[Service]
Environment=GOMAXPROCS=2
Restart=on-failure
ExecStart=/usr/local/bin/fabio
ExecReload=/bin/kill -HUP $MAINPID
KillSignal=SIGTERM

[Install]
WantedBy=multi-user.target

```

至此，我们的fabio+consul+registrator就算是部署完成了，那么我们怎么使用呢，这里，我们以`kubernetes-dashboard`为例

**kubernetes-dashboard.yaml**

```yaml
apiVersion: v1
kind: Secret
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard-certs
  namespace: kube-system
type: Opaque

---
apiVersion: v1
kind: ServiceAccount
metadata:
  labels:
    k8s-app: kubernetes-dashboard
  name: kubernetes-dashboard
  namespace: kube-system

---
# ------------------- Dashboard Role & Role Binding ------------------- #

kind: Role
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: kubernetes-dashboard-minimal
  namespace: kube-system
rules:
  # Allow Dashboard to create and watch for changes of 'kubernetes-dashboard-key-holder' secret.
  - apiGroups: [""]
    resources: ["secrets"]
    verbs: ["create", "watch"]
    # Allow Dashboard to get, update and delete Dashboard exclusive secrets.
  - apiGroups: [""]
    resources: ["secrets"]
    resourceNames: ["kubernetes-dashboard-key-holder"]
    verbs: ["get", "update", "delete"]
    # Allow Dashboard to get and update 'kubernetes-dashboard-settings' config map.
  - apiGroups: [""]
    resources: ["configmaps"]
    resourceNames: ["kubernetes-dashboard-settings"]
    verbs: ["list","get", "update"]
    # Allow Dashboard to get metrics from heapster.
  - apiGroups: [""]
    resources: ["services"]
    resourceNames: ["heapster"]
    verbs: ["proxy"]

  ---
  apiVersion: rbac.authorization.k8s.io/v1beta1
  kind: RoleBinding
  metadata:
    name: kubernetes-dashboard-minimal
    namespace: kube-system
  roleRef:
    apiGroup: rbac.authorization.k8s.io
    kind: Role
    name: kubernetes-dashboard-minimal
  subjects:
  - kind: ServiceAccount
    name: kubernetes-dashboard
    namespace: kube-system

  ---
  # ------------------- Dashboard Deployment ------------------- #

  kind: Deployment
  apiVersion: extensions/v1beta1
  metadata:
    labels:
      k8s-app: kubernetes-dashboard
    name: kubernetes-dashboard
    namespace: kube-system
  spec:
    replicas: 1
    revisionHistoryLimit: 10
    selector:
      matchLabels:
        k8s-app: kubernetes-dashboard
    template:
      metadata:
        labels:
          k8s-app: kubernetes-dashboard
      spec:
        containers:
          - name: kubernetes-dashboard
            image: gcr.io/google_containers/kubernetes-dashboard-amd64:v1.7.1
            ports:
            - containerPort: 9090
              protocol: TCP
            # 这里个人添加一些必要的env
            env:
            - name: SERVICE_9090_CHECK_HTTP
              value: "/"
            - name: SERVICE_9090_CHECK_INTERVAL
              value: "15s"
            - name: SERVICE_9090_CHECK_TIME
              value: "1s"
            - name: SERVICE_NAME
              value: kubernetes-dashboard
            - name: SERVICE_TAGS
              value: urlprefix-dashboard.quark.com/
              # 指定获取pod ip
            - name: POD_IP
              valueFrom:
                fieldRef:
                  fieldPath: status.podIP
            args:
              # Uncomment the following line to manually specify Kubernetes API server Host
              # If not specified, Dashboard will attempt to auto discover the API server and connect
              # to it. Uncomment only if the default does not work.
              # - --apiserver-host=http://my-address:port
              - --authentication-mode=basic
              # 这里添加一个连接heapster
              - --heapster-host=http://heapster.kube-system.svc.cluster.local
              volumeMounts:
                # Create on-disk volume to store exec logs
              - mountPath: /tmp
                name: tmp-volume
              livenessProbe:
                httpGet:
                  path: /
                  port: 9090
                initialDelaySeconds: 30
                timeoutSeconds: 30
            volumes:
            - name: tmp-volume
              emptyDir: {}
              # 这里的serviceAccountName改成default
            serviceAccountName: default
            # Comment the following tolerations if Dashboard must not be deployed on master
            tolerations:
            - key: node-role.kubernetes.io/master
              effect: NoSchedule
```

**前端LB配置**

```bash
server {
    listen       80;
    server_name  dashboard.quark.com;
    client_max_body_size 0;

    location / {
        proxy_pass                          http://fabio-server;
        proxy_set_header  Host              dashboard.quark.com;   # 这个位置一定是注册到consul里面的tags部分
        proxy_set_header  X-Real-IP         $remote_addr; # pass on real client's IP


        proxy_set_header  X-Forwarded-For   $proxy_add_x_forwarded_for;
        proxy_set_header  X-Forwarded-Proto $scheme;
        proxy_read_timeout                  900;
    }

}
```

```bash
upstream fabio-server {
        server 172.29.151.4:9999;
}
```

最后看看结果

> fabio

![](/images/posts/fabio.png)


> kubernetes-dashboard

![](/images/posts/dashboard.png)
