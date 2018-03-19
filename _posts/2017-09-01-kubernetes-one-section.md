---
layout: post
title: kubernetes 入门
categories: [kubernetes,docker]
description: kubernetes 入门
keywords: kubernetes,docker
---

## 入门概念

为什么要使用kubernetes

1.新技术

2.精简.只需要一个架构师专注于“服务组件”的提炼，几名开发工程师专注于代码开发，几名系统运维工程师负责kubernetes的部署和运维

3.kubernetes使用微服务架构

4.更方便迁移

5.超强的横向扩容能力

<!--more-->

### master

* kube-apiserver
提供HTTP RESET接口的关键服务进程，是kubernetes所有资源增删改查的唯一入口，也是集群控制的入口进程

* kube-controller-manager
kubernetes里所有资源对象的自动化控制中心。

* kube-scheduler
kubernetes里所有资源的调度中心

* kube-proxy
实现kubernetes service的通信与负载均衡机制的重要组件

* kubelet
负责pod对应容器的创建，启停等任务，同时与Master节点密切协作，实现集群管理的基本功能

* etcd
保存kubernetes里所有数据的存储

* docker
运行kubernetes 里面的容器

### node

* kube-proxy
实现kubernetes service的通信与负载均衡机制的重要组件

* kubelet
负责pod对应容器的创建，启停等任务，同时与Master节点密切协作，实现集群管理的基本功能

* docker
运行kubernetes 里面的容器


### pod

一个pause容器和一组业务容器组成，是kubernetes里面最基本的单元。

#### pause 容器

* 既然pod是一组容器组成，那么如何来判断这个pod的状态呢，是其中一个容器死亡了，就算整个pod死亡了，还是说按照某种N/M的死亡率来算呢？kubernetes里面引入了一个不易死亡又和业务无关的`pause`容器，以它的状态来表示整个pod的状态。
* pod里面的多个容器共享`pause`容器里面的网络和volumes。

#### 普通pod和静态pod

pod有两种类型：普通pod和静态pod
* 普通pod即是那些通过deployment，replicationcontroller，daemonset等部署的，这些pod一旦创建会被放入到etcd中，然后会被kubernetes master调度到某个node上，通过node上的kubelet进程实例化成一组相关的容器并启动起来。
* 静态pod是通过放在某个node上的一个具体的文件运行起来的。比如我们放在/etc/kubernetes/manifests下的某些静态文件。

#### endpoints

kubernetes中，每个pod都有一个属于他的IP。`Pod IP + 需要暴露出来的ContainerPort` 就组成了endpoint
说到service的endpoint，这里就不得不说下targetPort，targetPort属性用来确定提供该服务的容器所暴露的端口号，如果你不指定targetPort，kubernetes默认使用你提供的port为targetPort

所以他们的关系应该是如下：

service 暴露一个提供服务的端口--->{pod IP+targetPort}(容器内部暴露出来提供服务的端口号)
                                         ||
                                     {endpoint}
#### pod volume

pod中的volume能够被pod中的多个容器访问。kubernetes中的volume与pod的生命周期相同，但与容器的生命周期不相关。
通常是声明一个volume，然后在容器中引用这个volume并mount到容器的某个目录上。
常用的类型有：
* emptyDir
* hostPath
* Persistent Volume(GCE Persistent Disks、NFS、RBD、ISCSCI、AWS ElasticBlockStore、GlusterFS)，这就设计到分布式存储和外部存储的一些操作了，后续再讲吧

```yaml
apiVersion: v1
kind: ReplicationController
metadata:
  name: myweb
spec:
  replicas: 5
  # selector 如果不指定，默认和.spec.template.labels的值相同
  selector:
    app: myweb
  template:
    metadata:
      labels:
        app: myweb
    spec:
      # 该处声明一个volume
      volumes:
      - name: datavol
        emptyDir: {}
      containers:
      - name: myweb
        image: kubeguide/tomcat-app:v1
        ports:
        - containerPort: 8080
        env:
        - name: MYSQL_SERVICE_HOST
          value: 'mysql-service'
        - name: MYSQL_SERVICE_PORT
          value: '3306'
        # 该处进行引用并挂在到容器内部
        volumeMounts:
        - name: datavol
          mountPath: /mydata-data
```

#### pod 资源限制

每个pod都能对其能使用的服务器上的资源来进行配额限制，当前能限制的只有CPU和Memory。在kubernetes里面，计算资源的限制主要是设定两个参数
* Limits 资源允许使用的最大值，不能超过
* Requests 资源允许使用的最小值，最少必须满足这个需求

我们通常将`request` 设置为容器平时正常运行时所需的资源，而将`Limits`设置为容器峰值负载情况下的最大使用量

#### label and label selector

label即标签，是Kubernetes系统中另一个核心概念。一个label是一个`key=value`的键值对。label可以附加到各种资源对象上，如Node、Pod、RC、Service等，一个资源对象可以附加无数的label，同一个label也可以附加到无数的资源对象上。

通过指定`label selector`来查询和筛选某些拥有label的资源对象，常用到的两种表达式
* 等式 (key=value)
* 集合式 (key in values)

`label selector`在kubernetes中常用的场景如下：
* kub-controller-manager通过在RC上定义的`label selector`来监控并控制POD的数量
* kube-proxy 通过service上的`label selector`来选择对应的pod，自动建立每个service到pod的路由请求，从而实现service的智能负载均衡
* 通过NodeSelector，将某些pod调度到指定的node上

**注意：我们在指定`label selector`时，需要和`.spec.template.metadata.labels`下的值相同，如果不指定`label selector`则默认保持和`.spec.template.metadata.labels`的值相同**


#### init容器

init容器在1.6版本已经推出了beta版本，但是以前的语法仍然被保留，未来可能会被抛弃掉，所以建议使用最新的语法。

init容器主要作用是为了执行一些在pod就绪前的一些初始化步骤，比如：

* 把一些安全级别的程序放在init container中执行，避免放在images中暴露
* 它们必须在应用程序启动之前完成，并且只有在init容器成功完成后，应用容器才能启动，这就提供了一种简单的阻塞或延迟应用容器启动的方法

1.6之后的写法如下：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp-pod
  labels:
    app: myapp
spec:
  # init 容器
  initContainers:
    - name: init-myservice
      image: busybox
      command: ['sh','-c','until nslookup myservice;do echo waiting for myservice; sleep 2;done;']
  # 业务容器
  containers:
  - name: myapp-container
    image: busybox
    command: ['sh','-c','echo The app is running! && sleep 3600']
```

### ReplicationController(RC)
我们将上面那个例子的yaml文件直接拿来解析
```yaml
# api版本，类型，全局唯一名称，这是所有kubernetes yaml文件都需要的
apiVersion: v1
kind: ReplicationController
metadata:
  name: myweb
# 定义pod期望数量
spec:
  replicas: 5
  # 用于筛选目标的selector 如果不指定，默认和.spec.template.labels的值相同
  selector:
    app: myweb
  # 当Pod副本数和期望数不一致时，用于创建新pod的模板
  template:
    metadata:
      labels:
        app: myweb
    spec:
      # 该处声明一个volume
      volumes:
      - name: datavol
        emptyDir: {}
      containers:
      - name: myweb
        image: kubeguide/tomcat-app:v1
        ports:
        - containerPort: 8080
        env:
        - name: MYSQL_SERVICE_HOST
          value: 'mysql-service'
        - name: MYSQL_SERVICE_PORT
          value: '3306'
        # 该处进行引用并挂在到容器内部
        volumeMounts:
        - name: datavol
          mountPath: /mydata-data
```

我们可以通过命令的形式来动态缩放POD的数量
```bash
kubectl scale rc mysql --replicas=2

# 删除pod
kubectl delete -f mysql-rc.yml
kubectl scale rc mysql --replicas=0
```

### ReplicaSet

ReplicaSet和ReplicationController唯一的区别就是:ReplicaSet支持集合式的`label selector`，我们平时很少单独使用`Replica Set`，它主要是被`Deployment`这个更高层的资源对象使用。

### Deployment
Deployment在内部使用`Replica Set`来实现目的，它管理着`Replica Set`，而它管理`Replica Set`的主要目的是为了支持版本回滚`Rollback`

从下面命令所展示出来的命名规则我们不难发现，`Deployment`创建的时候创建了`Replica Set`，而`Replica Set`创建了`Pod`
```bash
$ kubectl get deployment -l k8s-app=kube-dns -n kube-system
NAME       DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
kube-dns   2         2         2            2           6d

$ kubectl get replicaset -l k8s-app=kube-dns -n kube-system
NAME                  DESIRED   CURRENT   READY     AGE
kube-dns-1446441763   2         2         2         6d

$ kubectl get pods -l k8s-app=kube-dns -n kube-system
NAME                        READY     STATUS    RESTARTS   AGE
kube-dns-1446441763-0th37   3/3       Running   0          6d
kube-dns-1446441763-1w5gx   3/3       Running   0          6d
```

稍后，用实验来说明`Deployment`在滚动升级中的作用

### Pod滚动升级

在说Deployment的滚动升级之前，我们先来看看`ReplicationController`的滚动升级

`ReplicationController`的滚动升级和`Deployment`的滚动升级有所不同，命令都不一样，通过执行`kubectl rolling-update`来一键完成，该命令创建了一个新的RC，然后自动控制旧的RC中的POD副本数两逐渐减少到0，同时新RC中的POD数量从0逐步增加到目标值，最终实现POD的升级，滚动升级的配置文件必须满足如下三个条件：
* `metadata.name`必须和旧RC文件中的不同
* `spec.selector`至少有一个与旧RC的不同(**这里有个BUG，具体参考[这里](http://valleylord.github.io/post/201603-kubernetes-roll/)**)
* `metadata.namespace` 命名空间必须的是一样的

下面我们将上面的`myweb`进行一下升级，内容如下
```yaml
apiVersion: v1
kind: ReplicationController
metadata:
  name: myweb-v2
spec:
  replicas: 5
  selector:
    version: v2
    deployment: v2
    app: myweb
  template:
    metadata:
      labels:
        version: v2
        deployment: v2
        app: myweb
    spec:
      # 该处声明一个volume
      volumes:
      - name: datavol
        emptyDir: {}
      containers:
      - name: myweb
        image: kubeguide/tomcat-app:v2
        ports:
        - containerPort: 8080
        env:
        - name: MYSQL_SERVICE_HOST
          value: 'mysql-service'
        - name: MYSQL_SERVICE_PORT
          value: '13306'
        # 该处进行引用并挂在到容器内部
        volumeMounts:
        - name: datavol
          mountPath: /mydata-data
```

啊哦，升级的时候报错了
```bash
error: myweb-rc-update.yml must specify a matching key with non-equal value in Selector for myweb
See 'kubectl rolling-update -h' for help and examples.
```

没关系，既然用yaml文件无法升级，我们用命令的形式来试试
```bash
kubectl rolling-update myweb --image=kubeguide/tomcat-app:v2
```

OK，成了，开始升级了

```bash
# 我们发现旧的rc正在逐步减少，新的rc正在增多
$ kubectl get rc
NAME                                     DESIRED   CURRENT   READY     AGE
mysql                                    1         1         1         34m
myweb                                    2         2         2         19m
myweb-c1dc64330c885b62eca9fb5aafbfecc6   4         4         4         3m

# 旧的pods也正在一个个的被新的pod替换
$ kubectl get pods
NAME                                           READY     STATUS    RESTARTS   AGE
mysql-65lw6                                    1/1       Running   0          35m
myweb-c1dc64330c885b62eca9fb5aafbfecc6-88f8v   1/1       Running   0          4m
myweb-c1dc64330c885b62eca9fb5aafbfecc6-c97fh   1/1       Running   0          58s
myweb-c1dc64330c885b62eca9fb5aafbfecc6-ttlpm   1/1       Running   0          2m
myweb-c1dc64330c885b62eca9fb5aafbfecc6-wd6kj   1/1       Running   0          3m
myweb-c1dc64330c885b62eca9fb5aafbfecc6-zr57k   1/1       Running   0          1m
myweb-k9tsj                                    1/1       Running   0          20m

```

成功了～

```bash
# 成功后，将RC名称改为myweb，升级完成
Created myweb-c1dc64330c885b62eca9fb5aafbfecc6
Scaling up myweb-c1dc64330c885b62eca9fb5aafbfecc6 from 0 to 5, scaling down myweb from 5 to 0
Scaling myweb-c1dc64330c885b62eca9fb5aafbfecc6 up to 1
Scaling myweb down to 4
Scaling myweb-c1dc64330c885b62eca9fb5aafbfecc6 up to 2
Scaling myweb down to 3
Scaling myweb-c1dc64330c885b62eca9fb5aafbfecc6 up to 3
Scaling myweb down to 2
Scaling myweb-c1dc64330c885b62eca9fb5aafbfecc6 up to 4
Scaling myweb down to 1
Scaling myweb-c1dc64330c885b62eca9fb5aafbfecc6 up to 5
Scaling myweb down to 0
Update succeeded. Deleting old controller: myweb
Renaming myweb-c1dc64330c885b62eca9fb5aafbfecc6 to myweb
replicationcontroller "myweb" rolling updated
```

那么为什么，我们在上面用yaml文件无法升级呢？

经过一段时间的折腾，我发现，我旧的RC文件只有一个label，app=myweb;而当我用命令升级成功后,新的RC有了两个label，app=myweb和deployment=xxxxx
```bash
# 未升级之前的RC
$ kubectl get rc -o wide
NAME      DESIRED   CURRENT   READY     AGE       CONTAINER(S)   IMAGE(S)                  SELECTOR
mysql     1         1         1         48m       mysql          mysql                     app=mysql
myweb     5         5         5         8s        myweb          kubeguide/tomcat-app:v1   app=myweb

# 升级后的RC
$ kubectl get rc -o wide
NAME       DESIRED   CURRENT   READY     AGE       CONTAINER(S)   IMAGE(S)                  SELECTOR
mysql      1         1         1         43m       mysql          mysql                     app=mysql
myweb      4         4         4         6m        myweb          kubeguide/tomcat-app:v2   app=myweb,deployment=c1dc64330c885b62eca9fb5aafbfecc6
```

然后当我再次对这个新的RC进行升级的时候，我发现是可以用yaml文件升级的，这说明什么？是不是旧的RC文件至少需要两个label才能用yaml文件升级呢？我们试试再说

* 为myweb添加不少于一个的label
```yaml
# api版本，类型，全局唯一名称，这是所有kubernetes yaml文件都需要的
apiVersion: v1
kind: ReplicationController
metadata:
  name: myweb
# 定义pod期望数量
spec:
  replicas: 5
  # 用于筛选目标的selector 如果不指定，默认和.spec.template.labels的值相同
  selector:
    app: myweb
    version: v1
  # 当Pod副本数和期望数不一致时，用于创建新pod的模板
  template:
    metadata:
      labels:
        app: myweb
        version: v1
    spec:
      # 该处声明一个volume
      volumes:
      - name: datavol
        emptyDir: {}
      containers:
      - name: myweb
        image: kubeguide/tomcat-app:v1
        ports:
        - containerPort: 8080
        env:
        - name: MYSQL_SERVICE_HOST
          value: 'mysql-service'
        - name: MYSQL_SERVICE_PORT
          value: '3306'
        # 该处进行引用并挂在到容器内部
        volumeMounts:
        - name: datavol
          mountPath: /mydata-data
```

* 新建一个用来升级myweb的yaml文件

```yaml
apiVersion: v1
kind: ReplicationController
metadata:
  name: myweb-v2
spec:
  replicas: 5
  selector:
    version: v2
    deployment: v2
    app: myweb
  template:
    metadata:
      labels:
        version: v2
        deployment: v2
        app: myweb
    spec:
      # 该处声明一个volume
      volumes:
      - name: datavol
        emptyDir: {}
      containers:
      - name: myweb
        image: kubeguide/tomcat-app:v2
        ports:
        - containerPort: 8080
        env:
        - name: MYSQL_SERVICE_HOST
          value: 'mysql-service'
        - name: MYSQL_SERVICE_PORT
          value: '13306'
        # 该处进行引用并挂在到容器内部
        volumeMounts:
        - name: datavol
          mountPath: /mydata-data
```

我发现，真的可以升级了!!!
```bash
$ kubectl rolling-update myweb -f myweb-rc-update.yml
Created myweb-v2
Scaling up myweb-v2 from 0 to 5, scaling down myweb from 5 to 0
Scaling myweb-v2 up to 1
Scaling myweb down to 4
Scaling myweb-v2 up to 2
Scaling myweb down to 3
Scaling myweb-v2 up to 3
Scaling myweb down to 2
Scaling myweb-v2 up to 4
```

**这说明什么？这说明，以后如果你要用`ReplicationController`的时候，至少要给他指定不少于2个的label，否则，你无法用yaml来进行升级，这真的是很蛋疼的一件事。**

当然了，保不齐以后kubernetes会将`ReplicationController`抛弃掉，毕竟现在它已经有了更好的`Deployment`和`StatefulSet`等...


### Horizontal Pod Autoscaler (HPA)

前面我们说到能用`kubectl scale`命令来手动实现 Pod 的扩容和缩容，这未免也太 low 了吧，我们既然用了kubernetes，那就是因为他的智能化，自动化。所以这里我们要讲讲kubernetes的智能扩容 HPA。

HPA 基于获取到的metrics value(CPU utilization,custom metrics),对RC,Deployment管理的pods进行自动伸缩。HPA是kubernetes `autoscaling` API组中的一个API资源，当前的stable版本只支持CPU，alpha版本中红，已经开始支持memory和custom metrics。

HPA 以kubernetes API resource 和一个controller来实现，resource决定了controller的行为，而controller控制着pods的数量。

> 截至到kubernetes 1.6 ，Release特性中仅支持CPU utilization这一 `resource metrics`, 对`custom metrics`的支持目前仍在alpha阶段。

HPA controller 周期性的调整对应rc，deployment中的pods数量，使得获取到的`metrics value`能匹配用户指定的`target utilization`。这个周期默认为30s，可以通过`kube-controller-manager`的flag `--horizontal-pod-autoscaler-sync-period`进行设置。

在每个HPA Controller的处理周期中，kube-controller-manager都去查询HPA获取到的metrics的utilization。查询方式根据metric类型不同而不同：

如果metric type是resource metrics，则通过resource metrics API查询，直接通过Heapster访问
如果metric type属于custom metrics，则通过custom metrics API查询，通过REST client来访问

计算伸缩比例算法：

* 对于 resource metrics,比如 CPU，HPA controller 从resource metrics API中获取CPU metrics，如果HPA中设定了target utilization，则HPA controller 会将获取到的CPU metrics 除以对应容器的resource request值作为检测到的当前pod的resource utilization。如此计算完所有HPA对应的pods后，对该resource utilization values取平均值。最后将平均值除与定义的target utilization，得到伸缩比例。

> 注意：如果HPA对应的某些pods中的容器没有定义resource request，则HPA不会对这些pods进行scale

* 对于custome metrics，HPA Controller的伸缩算法几乎与resource metrics一样，不同的是：此时是根据custome metrics API查询到的metrics value对比target metrics value计算得到的，而不是通过utilization计算得到的

* 对于object metrics，HPA Controller获取到一个metric 值，然后与target metrics比较，得到如上所说的比率

HPA可以通过命令来实现，也可以通过配置文件的方式来实现。

```bash
$ kubectl autoscale deployment php-apache --cpu-percent=50 --min=1 --max=10
deployment "php-apache" autoscaled
```

下面我们用实验来感受下HPA

首先我们来新建一个`php-apache`的服务

php-apache-deploy.yml
```yaml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: php-apache
  namespace: default
spec:
  replicas: 1
  selector:
    matchLabels:
      k8s-quark: php-apache
  template:
    metadata:
      labels:
        k8s-quark: php-apache
    spec:
      containers:
      - name: php-apache
        image: gcr.io/google_containers/hpa-example:latest
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 80
        resources:
          requests:
            cpu: 200m
```

php-apache-svc.yml
```yaml
apiVersion: v1
kind: Service
metadata:
  name: php-apache
spec:
  ports:
  - port: 80
  selector:
    k8s-quark: php-apache
```

```bash
kubectl create -f php-apache-deploy.yml
kubectl create -f php-apache-svc.yml

# 查看当前的deployment
$ kubectl get deployment php-apache
NAME         DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
php-apache   1         1         1            1           10m

# 查看php-apache的pods
$ kubectl get pods |grep php-apache
php-apache-3548797493-twq7k       1/1       Running   0          17m
```

然后，我们用命令来创建一个HPA
```bash
kubectl autoscale deployment php-apache --cpu-percent=50 --min=1 --max=10

# 看看我们创建好的hpa
$  kubectl get hpa
NAME         REFERENCE               TARGETS    MINPODS   MAXPODS   REPLICAS   AGE
php-apache   Deployment/php-apache   0% / 50%   1         10        1          19m

```
这条命令的意思是，我们为`php-apache`创建了一个HPA，指定了target metrics value是cpu利用率50%，而伸缩最小值为1，最大值为10

最后，我们来持续访问`php-apache`来给它压力，看看HPA会不会自动为我们扩容呢？

```bash
# 进入一个容器
kubectl run -ti load-generator --image=busybox /bin/sh

# 持续访问
while true; do wget -q -O- http://php-apache.default.svc.cluster.local; done
```

过了几分钟，我们看看结果咋样

```bash
# HPA状态
$  kubectl get hpa
NAME         REFERENCE               TARGETS      MINPODS   MAXPODS   REPLICAS   AGE
php-apache   Deployment/php-apache   313% / 50%   1         10        4          22m

# deployment状态
$  kubectl get deployment php-apache
NAME         DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
php-apache   4         4         4            4           25m

# HPA状态
$  kubectl get hpa
NAME         REFERENCE               TARGETS     MINPODS   MAXPODS   REPLICAS   AGE
php-apache   Deployment/php-apache   90% / 50%   1         10        8          26m

```

我们发现，随着我们的持续访问增压，HPA会自动的为我们将`php-apache`进行扩容，随着`php-apahce`的扩容，CPU开始慢慢下降，直到最终符合我们指定的低于50%的标准，或者达到最大值10个POD。而当我们停止对`php-apache`的访问，最终，HPA会恢复到默认1个pod的状态。

用yaml文件的方式，最终的效果和上面是一样的
```yaml
apiVersion: autoscaling/v1
kind: HorizontalPodAutoscaler
metadata:
  name: php-apache
  namespace: default
spec:
  # 指定针对谁来使用HPA
  scaleTargetRef:
    apiVersion: extensions/v1beta1
    kind: Deployment
    name: php-apache
  minReplicas: 1
  maxReplicas: 10
  # 用户定义的CPU利用率
  targetCPUUtilizationPercentage: 50
```

至于其它的HPA metrica，改天再讲吧，毕竟现在还只是alpha版本，而且需要heapster目前也无法收集那么多metrics

### Service

![](/images/posts/service-rc-pod.png)

透过上图，我们可以发现，service定义了一个服务的访问入口地址，前端的pod通过这个入口地址来访问其背后一组由pod组成的集群实例，而背后这组pod则是通过RC来生成并保持住的。他们三者之间，通过`label selector`来保持关联。

我们知道，正常情况下，要通过一个如果访问后端的集群服务，最好的办法是在前端弄一个负载均衡(nginx,haproxy...)，暴露一个对外服务的端口，然后反代到后端的ip+port。而kubernetes也遵循了这样的做法。
在上图中，frontal pod 访问 service 时，kubernetes其实是通过其内部的 kube-proxy 来进行负载均衡，然后将请求转发到后端的某个pod上。但kubernetes不是使用的一个实际的负载均衡IP地址，而是为每个service分配了一个全局唯一的虚拟IP地址，这个虚拟IP被称为Cluster IP，只能在kubernetes 集群内部被访问(意思就是只能在集群内的pod中才能访问)，而且一旦创建，在service的整个生命周期内，都不会发生变动。

下面我们来创建一个service看看
```yaml
apiVersion: v1
kind: Service
metadata:
  # service全局唯一名称，后面在cluster中可以直接使用的名称
  name: tomcat-service
spec:
  ports:
    # service提供服务的端口
  - port: 8080
    name: service-port
  # 后端容器提供服务暴露的端口，如果不指定，默认暴露service提供服务的端口
    targetPort: 8080
  - port: 8005
    name: shutown-port
    targetPort: 8005
  # 指定label selector 确认该服务和后端那些pod关联起来
  selector:
    app: mysql
```

#### kubernetes服务发现机制

最开始的时候kubernetes使用变量的形式，来发现服务，后来使用内部DNS来进行服务名解析，还有人认为使用consul的服务发现机制，其实我并不觉得这比内部的DNS服务发现好多少。

##### kubernetes内部dns寻址服务发现

1.kubectl 执行创建的时候会向APIServer请求创建一个service。APIServer获取到请求后调用相应的api创建一个service对象，并写入etcd保存。
2.kube-dns通过`list/watch`操作向APIServer发送GET请求。这时因为有service的创建，所以APIServer会相应这个请求并把service回复给kube-dns。
3.APIServer将创建的service信息回复给kube-dns,还会附带一个APIServer分配给service的Cluster IP。
4.kube-dns通过检测并得到APIServer回复的service信息，会生成DNS条目，并把这个DNS条目存储到内存(Tree-Cache)中
5.kubernetes中访问service的时候，会先去dnsmasq中查找缓存，找不到则去kubedns中查找dns条目，最终实现service的解析。(sidecar是用于检查其他两个容器的健康状态)

通过dns的服务发现机制，有个弊端就是服务的健康检查，不过这一点通过pod的健康检查可以填补。

##### consul、etcd等服务发现机制

consul：这个具体还没实施过，大致意思就是，容器启动时注册自己的ip+port到consul，然后consul自己做健康检查，最终将其发往fabio，fabio是个大路由，前端统一反代到fabio。
etcd：大体实现方式，就是写脚本通过etcd的api注册服务，然后再写一个service discover的脚本循环查询注册进去的service，对比template中的内容，然后生成新的配置文件进行更新。

#### kubernetes service 暴露

目前kubernetes service 暴露的方式有如下几种

* ClusterIP 只提供kubernetes集群内部的服务发现
* NodePort 在每个节点上提供端口暴露服务
* LoadBlancer 只能在云平台上使用，使用云平台提供的LB来暴露服务
* ExternalName 与另一个域名绑定，通过该service访问另一个服务
* ingress/traefik 使用第三方插件将pod暴露出来

而无论是ClusterIP 还是 NodePort 都是通过kube-proxy来对service进行实现的，而kube-proxy又有两种方式来实现负载，userspace和iptables，下面我来说一下kubernetes默认的iptables方式的kube-proxy

首先，我们来新建一个NodePort类型的服务

```yaml
apiVersion: v1
kind: Service
metadata:
  name: myweb-service
spec:
  type: NodePort
  ports:
  - port: 8080
    nodePort: 30001
  selector:
    app: myweb         
```

myweb-service 代理了后端的一个pod，ip为10.233.88.53，看看iptables

下面来逐条分析

如果是通过node的30001来访问，则会进入如下的链

```bash
# 看看和NodePort  30001有关的iptables

$ iptables -S -t nat |grep 30001
-A KUBE-NODEPORTS -p tcp -m comment --comment "default/myweb-service:" -m tcp --dport 30001 -j KUBE-MARK-MASQ
-A KUBE-NODEPORTS -p tcp -m comment --comment "default/myweb-service:" -m tcp --dport 30001 -j KUBE-SVC-KINM4OXG42E5QTAT
```

然后进一步跳转到`KUBE-SVC-KINM4OXG42E5QTAT`的链

```bash
-A KUBE-SVC-KINM4OXG42E5QTAT -m comment --comment "default/myweb-service:" -j KUBE-SEP-I4OJ7A6SXM5YG2QP
```

然后会跳转到`KUBE-SEP-I4OJ7A6SXM5YG2QP`链，最终将请求转发到10.233.88.53的pod上

```bash
-A KUBE-SEP-I4OJ7A6SXM5YG2QP -p tcp -m comment --comment "default/myweb-service:" -m tcp -j DNAT --to-destination 10.233.88.53:8080
```

> **注意：如果service代理了多个pod的话，会利用iptables的--probability特性，按一定的比例转发，如下**

假若我们的myweb-service代理了3个pod

如果是通过node的30001来访问，则会进入如下的链

```bash
# 看看和NodePort  30001有关的iptables

$ iptables -S -t nat |grep 30001
-A KUBE-NODEPORTS -p tcp -m comment --comment "default/myweb-service:" -m tcp --dport 30001 -j KUBE-MARK-MASQ
-A KUBE-NODEPORTS -p tcp -m comment --comment "default/myweb-service:" -m tcp --dport 30001 -j KUBE-SVC-KINM4OXG42E5QTAT
```

然后进一步跳转到`KUBE-SEP-5I5KUCBAI2CKFMN2`、`KUBE-SEP-I4OJ7A6SXM5YG2QP`、`KUBE-SEP-FDQHGZ7N6PHRDJRL`的链

```bash
# 有分为30%，50%，20%的几率
-A KUBE-SVC-KINM4OXG42E5QTAT -m comment --comment "default/myweb-service:" -m statistic --mode random --probability 0.33332999982 -j KUBE-SEP-5I5KUCBAI2CKFMN2

-A KUBE-SVC-KINM4OXG42E5QTAT -m comment --comment "default/myweb-service:" -m statistic --mode random --probability 0.50000000000 -j KUBE-SEP-I4OJ7A6SXM5YG2QP

-A KUBE-SVC-KINM4OXG42E5QTAT -m comment --comment "default/myweb-service:" -j KUBE-SEP-FDQHGZ7N6PHRDJRL
```

然后会分别跳到上面每个链下对应的链，最终转发到对应的pod上

```bash
# KUBE-SEP-5I5KUCBAI2CKFMN2
-A KUBE-SEP-5I5KUCBAI2CKFMN2 -p tcp -m comment --comment "default/myweb-service:" -m tcp -j DNAT --to-destination 10.233.87.108:8080

# KUBE-SEP-I4OJ7A6SXM5YG2QP
-A KUBE-SEP-I4OJ7A6SXM5YG2QP -p tcp -m comment --comment "default/myweb-service:" -m tcp -j DNAT --to-destination 10.233.88.53:8080

# KUBE-SEP-FDQHGZ7N6PHRDJRL
-A KUBE-SEP-FDQHGZ7N6PHRDJRL -p tcp -m comment --comment "default/myweb-service:" -m tcp -j DNAT --to-destination 10.233.96.254:8080
```

好了，说完了NodePort，然后我们再说ClusterIP

继续从上面看到尾就行了。

ingress和traefik留到后面单独留个篇幅来说吧，这里就不说了。

### Volume

kubernetes 的volume要说的内容就很多了。

首先volume是一个在pod中能被多个容器访问的共享目录，它是定义在pod上，然后挂载到容器下，而且它的生命周期只和pod有关。常见的存储类型有如下几种

* emptyDir
* hostPath
* GCE Persistent Disks
* NFS
* RBD
* ISCSCI
* AWS ElasticBlockStore
* GlusterFS
* secret
* PersistentVolume
* downwardAPI
* projected
* configmap
* local

**注意：gitRepo实际上也是挂载一个空目录，从GIT仓库中clone内容下来供pod使用，所以它的数据也无法永久保存**

#### emptyDir

正如它的名字一样，这是一个空的目录，它是在Pod被分配到node节点上的时候被创建的，无需在宿主机node上指定对应的目录文件。而且当pod从节点上被删除之后，emptyDir中的数据也会被删除，emptyDir 一般用在如下几个地方

* 暂存空间，例如用于基于磁盘的归并排序或长计算的检查点的暂存空间
* 临时目录，临时储存那些无需持久化的数据

默认情况下，`emptyDir` 数据存储在SSD或者网络存储上。但是，你也可以设置`emptyDir.medium`为`Memory`来启用`tmpfs`,tmpfs会将数据写入到内存中，因此，当机器重启之后，数据也会永久删除，并且，因为是存放在内存中，这些数据会占用你容器内存的limit指标。

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-pd
spec:
  containers:
  - image: nginx
    name: test-container
    volumeMounts:
    - mountPath: /cache
      name: cache-volume
  volumes:
  - name: cache-volume
    emptyDir:
      medium: Memory
```

#### secret

secret volume 使用来传递敏感信息，比如说密码之类的。我们可以将在kubernetes中定义的secret直接挂载为文件让pod访问。secret volume实际是通过tmpfs(内存文件系统)来实现的，所以这些信息不会持久保存。

#### downwardAPI

如果你想将pod或container里面的字段暴露给其他正在运行的容器，那么downwardAPI正是你所需要的，它会将pod或container里面的字段以文件的形式存储下来，然后挂载到对应的容器中。

目前能暴露的字段
* node's name
* pod's name
* pod's namespace
* pod's ip
* pod's serviceAccount name
* pod's UID
* pod's labels
* pod's annotations
* 容器 CPU limit
* 容器 CPU request
* 容器 memory limit
* 容器 memory request

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: kubernetes-downwardapi-volume-example
  labels:
    zone: us-est-coast
    cluster: test-cluster1
    rack: rack-22
  annotations:
    build: two
    builder: john-doe
spec:
  containers:
    - name: client-container
      image: gcr.io/google_containers/busybox
      command: ["sh", "-c"]
      args:
      - while true; do
          if [[ -e /etc/labels ]]; then
            echo -en '\n\n'; cat /etc/labels; fi;
          if [[ -e /etc/annotations ]]; then
            echo -en '\n\n'; cat /etc/annotations; fi;
          sleep 5;
        done;
      volumeMounts:
        - name: podinfo
          mountPath: /etc
          readOnly: false
  volumes:
    - name: podinfo
      downwardAPI:
        items:
          - path: "labels"
            fieldRef:
              fieldPath: metadata.labels
          - path: "annotations"
            fieldRef:
              fieldPath: metadata.annotations
```

#### projected

projected volume 可以将几个volume内容隐射到同样的目录中，当前只支持secret、configmap、downwardAPI

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: volume-test
spec:
  containers:
  - name: container-test
    image: busybox
    volumeMounts:
    - name: all-in-one
      mountPath: "/projected-volume"
      readOnly: true
  volumes:
  - name: all-in-one
    projected:
      sources:
      - secret:
          name: mysecret
          items:
            - key: username
              path: my-group/my-username
      - downwardAPI:
          items:
            - path: "labels"
              fieldRef:
                fieldPath: metadata.labels
            - path: "cpu_limit"
              resourceFieldRef:
                containerName: container-test
                resource: limits.cpu
      - configMap:
          name: myconfigmap
          items:
            - key: config
              path: my-group/my-config
```

#### local

local volume在1.7中目前还是alpha版本，主要是用来将本地的disk，分区或者目录进行挂载。local volume只能以静态创建的PV使用。相对于HostPath，localhost可以直接以持久化的方式使用（它总是通过NodeAffinity调度在某个指定的节点上），而hostpath是无法直接以pv来使用的。

#### hostPath

hostPath 就是将宿主机上的目录或文件挂在到pod里。比如我们常用到`hostPath`的几个例子

* 容器应用程序的日志，需要永久保存时，可以使用hostPath隐射宿主机上的高速文件存储
* 运行的容器需要访问Docker内部结构：使用hostPath映射/var/lib/docker
* 在容器中运行cAdvisor，使用hostPath映射/dev/cgroups

而当我们在使用hostPath的时候需要注意以下几点

* 在不同Node上具有相同配置的pod(通过podTemplate创建的)，可能会因为宿主机上的目录和文件不同而导致volume上目录和文件的访问结果不一致
* 如果使用资源配额，无法将hostPath在宿主机上使用的资源纳入管理
* 如果宿主机上的目录是root权限，那么你也必须以root身份来运行你的进程，或者，修改你的目录权限以便于能让hostPath卷有写权限

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-pd
spec:
  containers:
  - image: nginx
    name: test-container
    volumeMounts:
    - mountPath: /test-pd
      name: test-volume
  volumes:
  - name: test-volume
    hostPath:
      path: /data
```



其实hostPath也能算是持久存储的一种，只不过局限性太大了。这里我们详细的讲讲外部存储和分布式存储在kubernetes中的使用(aws和gce的就不讲了，没环境)。

[详细的例子](https://github.com/kubernetes/kubernetes/tree/master/examples/volumes)大家可以参考官方的例子

#### NFS

NFS是Network File System的缩写，即网络文件系统。kubernetes中通过简单的配置就可以挂载NFS到Pod中，而NFS中的数据是可以永久保存的，同时NFS支持同时写操作。

首先，你得有个已经搭建好的NFS服务

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-pd
spec:
  containers:
  - image: nginx
    name: test-container
    volumeMounts:
    - mountPath: /test-pd
      name: test-volume
  volumes:
  - name: test-volume
    nfs:
      server: 163.xx.xx.xx
      path: "/"
```

#### persistent volume

事实上，我们可以单独在Pod中指定外部存储，也可以将这些外部存使用PersistentVolume资源化。

之前我们提到的volume都是定义在pod上的，和 pod 是一种静态绑定关系，属于"计算资源"的一部分，而实际上，“网络存储”是独立于“计算资源”之外而存在的一种实体资源。比如我们在使用虚机的情况下，我们通常会定义一个网络存储，然后从中划出一定的空间连接到虚拟机。而persistent volume和与之关联的persistent volume claim就起到了类似的作用。

PersistentVolume(PV)是集群中的一块网络存储，用来提供网络存储资源 。跟Node一样，也是集群的资源。PV 跟volume类似，不过会有独立于Pod的生命周期。
PersistentVolumeClaim(PVC)是对PV的请求。PVC有点类似于Pod，pod消耗node的资源，而PVC消耗PV的资源。Pod请求CPU和内存，而PVC请求特定大小和访问模式的数据卷。

PV的访问模式有三种：

* ReadWriteOnce(RWO):最基本的访问方式，可读可写，但只支持被耽搁pod挂载
* ReadOnlyMany(ROX):可以以只读的方式被多个pod挂载
* ReadWriteMany(RWX):以读写的方式被多个POD挂载。

并不是所有存储都支持这三种方式，像共享方式，目前支持的还比较少，比较常用的是NFS。在PVC绑定PV时通常根据两个条件来绑定，一个实存储大小，一个是访问方式。
**需要注意的是，虽然PV支持三种访问模式，但它同时只支持一种方式来访问PV**


##### 创建PV的方式有两种

有两种创建PV的方式：静态和动态

###### 静态

所谓静态，就是管理员手动创建一堆PV，组成一个PV池，供PVC来绑定。

###### 动态
经过API抽象，用户可以通过PVC使用存储资源，通常用户还会关心PV的很多属性，例如对不同的应用场景需要不同的性能，仅仅提供存储大小和访问模式不能满足要求。集群管理员一方面要提供不同PV的多种属性，一方面要隐藏底层的细节，还有一点是不再需要管理员手动去创建PV,这就引入了`StorageClass`资源。管理员用存储级别StorageClass描述存储的分类，不同的分类可以对应不同的质量服务Qos等级、备份策略和其他自定义的策略。kubernetes本身不参与存储级别的划分，StorageClass概念在有的存储系统里被称为”profiles”

所谓动态，就是当所有的静态PV都不匹配用户的PVC时，集群通过storageClass的对象由存储系统根据PVC的要求自动创建。这种基于`StorageClass`的PV，管理员必须事先创建和配置这样的storage class。请求等级配置为`" "`的PVC，有效地禁用了它自身的动态供给功能。

###### class

我们可以根据不同的需求创建不同类型的PV(这种PV是基于`StorageClass`，是自动创建的)，然后我们可以通过为PVC指定`storageClassName`来请求PV，如果集群中有这个class，那么当用户请求的时候，kubernetes会自动的创建PV，如果集群中有默认的storageclass，那么你只需要创建PVC即可，无需指定`storageClassName`,剩下的都有默认的动态配置来搞定。

举个例子：比如我现在需要两种类型的存储，一种是SSD，一种是普通的硬盘，那么这时候，我就可以创建两种class的`StorageClass`,然后在创建PVC时，指定不同的`storageClassName`即可，如下：

每个`StorageClass`都包含`provisioner`和`parameters`这个两个字段，具体怎么配置这些storageclass，有哪些存储支持storageclass,请参考[官网](https://kubernetes.io/docs/concepts/storage/persistent-volumes/)


```yaml
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: slow
provisioner: kubernetes.io/gce-pd
parameters:
  type: pd-standard
```

```yaml
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: fast
provisioner: kubernetes.io/gce-pd
parameters:
  type: pd-ssd
```

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-fast
  storageClassName: fast
spec:
  accessModes:
    - ReadOnlyMany
  resources:
    requests:
      storage: 2Gi
```
当用户请求资源的时候，如果是通过pvc-fast请求，这时候就会绑定到一个SSD，而不会绑定到普通硬盘。

###### 绑定

在动态配置的情况下，当用户创建或者之前就已经创建了具有特定数量或具有某些访问模式的PVC的时候。master中的loop control会监视新的PVC，找到匹配的PV，然后将它们绑定到一起。如果这个PV是通过动态提供给PVC，那么loop control会始终绑定这个PV给这个PVC。否则，用户可能会一直请求PV，但是实际上又没有那么多PV资源。还需要注意的事，一旦绑定了PVC，就不能再绑定其他。

###### 使用

Pod使用PVC和使用Volume一样。集群检查PVC然后找到绑定的pv，然后隐射给pod使用。
一旦用户拥有了一个PVC，并且PVC被绑定，那么只要用户还需要，PV就一直属于这个用户。

###### 回收

当用户不在需要PV时，我们可以删除PVC来回收PV，`PersistentVolume`中的回收策略会告诉kubernetes当PVC被释放后该怎么做，目前，支持的策略如下

* Retained(保留)

当PVC被删除后，PV仍然被保留下来，并且会变成`released`。但是它还不能被其他PVC使用，因为现在PV上仍然有上一个PVC所请求的数据。

* Recycled(再利用)

当PVC被删除后，kubernetes会将PV里的数据删除，然后把PV变成Available，然后又可以被新的PVC绑定

这个原理实际上是：在删除PVC之后，会运行一个POD来执行一个(`rm -rf /thevolume/*`)的操作，删除pv下的所有数据

**默认回收运行的POD用的image是gcr上的busybox，而且image策略是always，因为这个原因，你可能始终无法回收PV，这时候，就需要去重新配置回收POD的模板了，模板内容如下**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: recycler-for-nfs
  namespace: default
spec:
  restartPolicy: Never
  volumes:
  - name: vol
    nfs:
      path: /
      server: 163.xx.xx.xx
  containers:
  - name: pv-recycler
    image: "gcr.io/google_containers/busybox"
    imagePullPolicy: IfNotPresent
    command: ["/bin/sh", "-c", "test -e /scrub && rm -rf /scrub/..?* /scrub/.[!.]* /scrub/*  && test -z \"$(ls -A /scrub)\" || exit 1"]
    volumeMounts:
    - name: vol
      mountPath: /scrub
```

写好模板了，还需要在`kube-controller-manager`中去配置模板，[更多详细信息](https://kubernetes.io/docs/admin/kube-controller-manager/)可参考官网

```yaml
# 指定nfs回收模板的位置
- --pv-recycler-pod-template-filepath-nfs=/etc/kubernetes/recycler-for-nfs.yaml

# 在容器下挂载模板文件
volumeMounts:
- mountPath: "/etc/kubernetes/recycler-for-nfs.yaml"
  name: recycler-nfs

# 将宿主机上的模板文件挂载到pod
volumes:
- name: recycler-nfs
  hostPath:
    path: "/etc/kubernetes/recycler-for-nfs.yaml"
```

下面我们来新建PV和PVC来测试一下

nfs-pv.yaml
```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: nfs
spec:
  capacity:
    storage: 5Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Recycle
  nfs:
    path: "/"
    server: 163.44.165.142
```

nfs-pvc.yaml
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: nfs
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
```

nfs-nginx.yaml
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-pd
spec:
  containers:
  - image: nginx
    name: test-container
    volumeMounts:
    - mountPath: /test-pd
      name: test-volume
  volumes:
  - name: test-volume
    persistentVolumeClaim:
      claimName: nfs

```


命令如下：

```bash
# 创建PV,PVC以及关联的POD
kubectl create -f nfs-pv.yaml
kubectl create -f nfs-pvc.yaml
kubectl create -f nfs-nginx.yaml

# 查看创建的PV
$ kubectl get pv
NAME      CAPACITY   ACCESSMODES   RECLAIMPOLICY   STATUS    CLAIM         STORAGECLASS   REASON    AGE
nfs       5Gi        RWO           Recycle         Bound     default/nfs                            1h

# 查看创建的PVC
$ kubectl get pvc
NAME      STATUS    VOLUME    CAPACITY   ACCESSMODES   STORAGECLASS   AGE
nfs       Bound     nfs       5Gi        RWO                          8m

# 然后我们进入pod中新建一个文件recycle-file
$ kubectl exec -ti test-pd /bin/bash
$ touch recycle-file
```

我们发现nfs的回收策略是`Recycle`，当前状态是`Bound`，那么假如我现在将PVC删除掉呢，下面我们来操作试试看

```bash
kubectl delete -f nfs-nginx.yaml
kubectl delete -f nfs-pvc.yaml

# 查看现在的pv
$ kubectl get pv
NAME      CAPACITY   ACCESSMODES   RECLAIMPOLICY   STATUS     CLAIM         STORAGECLASS   REASON    AGE
nfs       5Gi        RWO           Recycle         Released   default/nfs                            1h

# 过了一会儿，再看pv
$ kubectl get pv
NAME      CAPACITY   ACCESSMODES   RECLAIMPOLICY   STATUS      CLAIM     STORAGECLASS   REASON    AGE
nfs       5Gi        RWO           Recycle         Available                                      1h

# 再重新使用PVC关联POD
kubectl create -f nfs-pvc.yaml
kubectl create -f nfs-nginx.yaml

# 进入pod中看看数据是否还存在
$ kubectl exec -ti test-pd ls /test-pd
```

我们发现当我们的回收策略是`Recycle`时，删除PVC之后，PV的状态先是`Released`，然后过一会儿之后，会变成`Available`状态，而且PV上的数据也已经被删除了，这时候就可以再次被其他的PVC使用了。

**注意：当前只有NFS和HostPath支持回收利用操作**

* Delete(删除)

当PVC被删除后，kubernetes会删除PV及里面的数据。

**注意：当前只有AWS EBS,GCE PD,AZURE DISK,OPENSTACK CINDER卷支持删除操作**

**注意：动态PV，总会在PVC被删除后被删除**

###### PV阶段状态

一个volume会处于下面的几个状态之一

* Avaliable 尚未绑定到PVC上的可用资源
* Bound 已经被绑定到PVC
* Released PVC已被删除，但是资源尚未回收
* Failed 自动回收失败

###### Capacity

通常，我们在创建PV的时候，会从存储上给它划定一定大小的容量，这就使用capacity来指定即可。

###### resource

pvc，就像pod一样，可以指定request资源的大小

###### selector

PVC也可以指定标签选择器进行深度过滤PV，只有匹配了selector的PV才能绑定给PVC

* matchLabels 单个匹配
* matchExpressions 表达式匹配

### Namespace

Namespace一般用于实现多租户的资源隔离。namespace 通过将集群内部的资源分配到不同的Namespace中，形成逻辑上不同项目、小组或环境的隔离，同时利用resource quota实现资源的管控限制，而随着kubernetes访问控制的深入，namespace开始与kubernetes的认证和授权机制结合。

### Annotation

annotation 和label类似，也是使用 key/value的形式进行定义。不同的是label具有严格的命名规则，它定义的是kubernetes对象的元数据，并且用于label selector，而annotation测试用户任意定义的"附加"信息，以便于外部工具进行查找，很多时候，kubernetes的模块会通过annotation的方式标记资源的特殊信息。
