---
layout: post
title: kubernetes ceph 笔记 3
categories: [kubernetes,ceph]
description: kubernetes ceph
keywords: kubernetes,ceph
---
> 前面花了两章的时间介绍了ceph存储集群，简单的讲了ceph的组件、架构、寻址过程以及关于rbd,cephfs,cephGW,rados,osd,mon,mds,pool,pg,object等的操作过程，这一章主要记录下kubernetes使用ceph的相关配置过程。


<!--more-->

通过[官网](https://kubernetes.io/docs/concepts/storage/persistent-volumes/#ceph-rbd)，我们发现在kubernetes中使用的是ceph的RBD

### 部署集群

具体的集群部署这里就不在赘述，请参考[kubernetes ceph 笔记 1](https://kevinguo.me/2017/09/06/kubernetes-ceph-1/)、[kubernetes ceph 笔记 2](https://kevinguo.me/2017/09/12/kubernetes-ceph-2/)，这里我们只是额外添加一组实验所需的osd，命令如下

##### 添加osd

为每台机器添加一个sdc的硬盘，我这里用目录代替硬盘

```bash
# 在每台osd节点上执行
sudo mkdir -p /data/sdc
chown ceph.ceph /data/sdc

#在管理节点上执行`ceph-deploy`来准备OSD
ceph-deploy osd prepare k8s-master01:/data/sdc k8s-master02:/data/sdc k8s-master03:/data/sdc k8s-node01:/data/sdc k8s-node02:/data/sdc k8s-registry:/data/sdc

# 激活OSD
ceph-deploy osd activate k8s-master01:/data/sdc k8s-master02:/data/sdc k8s-master03:/data/sdc k8s-node01:/data/sdc k8s-node02:/data/sdc k8s-registry:/data/sdc

# 检测集群状态
ceph halth
```

##### 创建RBD

```bash
# 创建存储池
rados mkpool data

# 创建image
rbd create data --size 10G -p data

# 关闭不支持特性
rbd feature disable data exclusive-lock, object-map, fast-diff, deep-flatten -p data

# 映射image到块设备(每个节点都需要隐射)
rbd map data --name client.admin -p data

# 格式化块设备
mkfs.xfs /dev/rbd0
```

### Kubernetes 使用Ceph

#### PV & PVC方式

传统使用分布式存储的方式一般为 `PV & PVC` 的方式，也就是说管理员必须预先创建好PV 和 PVC ，然后对应的deployment或者replication挂载PVC来使用

##### 创建secret

```bash
# 获取管理 key 并进行 base64 编码
ceph auth get-key client.admin | base64

# 创建一个secret 配置(key 为上一条命令生成)
vim ceph-secret.yml

apiVersion: v1
kind: Secret
metadata:
  name: ceph-secret
data:
  key: QVFCZmVyWlpFS1hGTHhBQWhsekVscG0yTWhoYkJHQjRUbk5Wa0E9PQ==

# 创建secret
kubectl create -f ceph-secret.yml
```

##### 创建PV

```bash
#创建ceph-pv文件
vim ceph-pv.yml

apiVersion: v1
kind: PersistentVolume
metadata:
  name: ceph-pv
spec:
  capacity:
    storage: 2Gi
  accessModes:
    - ReadWriteMany
  rbd:
    monitors:
      - 172.30.33.90:6789
      - 172.30.33.91:6789
      - 172.30.33.92:6789
    pool: data
    image: data
    user: admin
    secretRef:
      name: ceph-secret
    fsType: xfs
    readOnly: false
  persistentVolumeReclaimPolicy: Recycle

# 创建PV
kubectl create -f ceph-pv.yml
```

##### 创建PVC

```bash
# 新建ceph-pvc.yml文件
vim ceph-pvc.yml

apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: ceph-pvc
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 2Gi

# 创建PVC
kubectl create -f ceph-pvc.yml
```

##### 创建一个测试的deployment来挂载

```bash
# 新建nginx.yml
apiVersion: apps/v1beta1
kind: Deployment
metadata:
  name: ceph-nginx
spec:
  replicas: 3
  template:
    metadata:
      labels:
        app: ceph-nginx
    spec:
      containers:
      - name: ceph-nginx
        image: nginx
        ports:
        - containerPort: 80
        volumeMounts:
          - mountPath: "/data"
            name: data
      volumes:
        - name: data
          persistentVolumeClaim:
            claimName: ceph-pvc

# 创建nginx deployment
kubectl create -f ceph-nginx.yml
```

##### 配置k8s node

我们创建好PV 和 PVC之后，进行查看时可能会出现`with: rbd: failed to modprobe rbd error:exit status 1`的报错，所以这时候我们需要对所有k8s-node进行如下配置

```bash
# 在所有k8s node上安装ceph-common
yum install -y ceph-common

# 拷贝ceph.conf和ceph.client.admin.keyring到/etc/ceph/目录下

# 配置kubelet有关ceph的参数，增加如下内容
vim /usr/local/bin/kubelet

-v /sbin/modprobe:/sbin/modprobe:ro \
-v /lib/modules:/lib/modules:ro \
-v /etc/ceph:/etc/ceph:ro \

# 重启kubelet
systemctl restart kubelet

# 查看pod是否启动成功
kubectl get pods
NAME                          READY     STATUS    RESTARTS   AGE
ceph-nginx-2497831062-569lw   1/1       Running   0          15m
ceph-nginx-2497831062-589j9   1/1       Running   0          59m
ceph-nginx-2497831062-5t01s   1/1       Running   0          12m

# 然后进入其中一个pod，写入一个1G的文件
kubectl exec -ti ceph-nginx-2497831062-569lw  /bin/bash

dd if=/dev/zero of=test-file bs=1G count=1

# 然后查看是否已经占用了rbd中的空间呢
ceph df
GLOBAL:
    SIZE     AVAIL     RAW USED     %RAW USED
    575G      344G         230G         40.09
POOLS:
    NAME                      ID     USED      %USED     MAX AVAIL     OBJECTS
    data                      14     1038M      1.50        68277M         280

# 然后我们删除这个pod
kubectl delete pods ceph-nginx-2497831062-569lw

# 查看新的pod,发现文件依旧在
kubectl exec -ti ceph-nginx-2497831062-rgkcl ls /data
test-file
```


#### StorageClass方式之StatefulSet

重头戏来了，洋洋洒洒写了近3篇文章，最终就是要使用这个StorageClass这个东西；这个东西在前面的[kubernetes入门](https://kevinguo.me/2017/09/01/kubernetes-one-section/#%E5%8A%A8%E6%80%81)有简单的提到过，就是说动态创建PV，不用再事先固定PV的大小，直接创建PVC即可分配使用。

##### 创建secret

```bash
# 获取管理 key 并进行 base64 编码
ceph auth get-key client.admin | base64

# 创建一个secret 配置(key 为上一条命令生成)
vim ceph-storageclass-secret.yml

apiVersion: v1
kind: Secret
metadata:
  name: ceph-storageclass-secret
data:
  key: QVFCZmVyWlpFS1hGTHhBQWhsekVscG0yTWhoYkJHQjRUbk5Wa0E9PQ==
type: kubernetes.io/rbd

# 创建一个namespace的secret
vim ceph-storageclass-secret-system.yml
apiVersion: v1
kind: Secret
metadata:
  name: ceph-storageclass-secret
  namespace: kube-system
data:
  key: QVFCZmVyWlpFS1hGTHhBQWhsekVscG0yTWhoYkJHQjRUbk5Wa0E9PQ==
type: kubernetes.io/rbd

# 创建secret
kubectl create -f ceph-storageclass-secret.yml
kubectl create -f ceph-storageclass-secret-system.yml
```

##### 创建一个storageclass

```bash
# 新建ceph-storageclass.yml文件
vim ceph-storageclass.yml

apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: ceph-storageclass
provisioner: kubernetes.io/rbd
parameters:
  monitors: 172.30.33.90:6789,172.30.33.91:6789,172.30.33.92:6789
  adminId: admin
  adminSecretName: ceph-storageclass-secret
  adminSecretNamespace: kube-system
  pool: data
  userId: admin
  userSecretName: ceph-storageclass-secret

# 新建storageclass
kubectl create -f ceph-storageclass.yml
```

##### 创建statefulset

我们在使用StorageClass的时候，可以自己手动创建PVC，然后所有pods共享一个pvc；也可以定义`volumeClaimTemplates`来为自动为每个pod创建一个单独的pvc，如下所示

```bash
# 新建ceph-storageclass-nginx.yml
vim ceph-storageclass-nginx.yml

apiVersion: apps/v1beta1
kind: StatefulSet
metadata:
  name: ceph-storageclass-nginx
spec:
  serviceName: "ceph-storageclass-nginx-service"
  replicas: 3
  template:
    metadata:
      labels:
        app: ceph-storageclass-nginx
    spec:
      containers:
      - name: ceph-storageclass-nginx
        image: nginx
        ports:
        - containerPort: 80
        volumeMounts:
          - mountPath: "/data"
            name: data
  volumeClaimTemplates:
  - metadata:
      name: data
      annotations:
        volume.beta.kubernetes.io/storage-class: ceph-storageclass-pvc
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 5Gi

# 新建ceph-storageclass-nginx-service.yml
vim ce ph-storageclass-nginx-service.yml

apiVersion: v1
kind: Service
metadata:
  name: ceph-storageclass-nginx-service
  labels:
    app: ceph-storageclass-nginx-service
spec:
  ports:
  - port: 80
    name: web
  clusterIP: None
  selector:
    app: ceph-storageclass-nginx

# 创建statefulSet
kubectl create -f ceph-storageclass-nginx.yml
kubectl create -f ceph-storageclass-nginx-service.yml

# 查看pods
kubectl get pods
NAME                          READY     STATUS    RESTARTS   AGE
ceph-storageclass-nginx-0     1/1       Running   0          11m
ceph-storageclass-nginx-1     1/1       Running   0          11m
ceph-storageclass-nginx-2     1/1       Running   0          11m

# 查看自动创建的pv和pvc
kubectl get pv
NAME                                       CAPACITY   ACCESSMODES   RECLAIMPOLICY   STATUS    CLAIM                                    STORAGECLASS        REASON    AGE
pvc-1af65cba-9dca-11e7-84a9-00155d201312   5Gi        RWO           Delete          Bound     default/data-ceph-storageclass-nginx-0   ceph-storageclass             15m
pvc-934ed5ad-9dca-11e7-84a9-00155d201312   5Gi        RWO           Delete          Bound     default/data-ceph-storageclass-nginx-1   ceph-storageclass             12m
pvc-9ff7359d-9dca-11e7-84a9-00155d201312   5Gi        RWO           Delete          Bound     default/data-ceph-storageclass-nginx-2   ceph-storageclass             11m


kubectl get pvc
NAME                             STATUS    VOLUME                                     CAPACITY   ACCESSMODES   STORAGECLASS        AGE
data-ceph-storageclass-nginx-0   Bound     pvc-1af65cba-9dca-11e7-84a9-00155d201312   5Gi        RWO           ceph-storageclass   15m
data-ceph-storageclass-nginx-1   Bound     pvc-934ed5ad-9dca-11e7-84a9-00155d201312   5Gi        RWO           ceph-storageclass   12m
data-ceph-storageclass-nginx-2   Bound     pvc-9ff7359d-9dca-11e7-84a9-00155d201312   5Gi        RWO           ceph-storageclass   11m

# 进入pod查看使用情况，发现/data使用大小5G
[root@k8s-master01 k8s-quark]# kubectl exec -ti ceph-storageclass-nginx-1 -- df -Th
Filesystem                                                                                      Type   Size  Used Avail Use% Mounted on
/dev/rbd1                                                                                       ext4   4.8G   10M  4.6G   1% /data
```

#### StorageClass方式之Deployment

我们接着使用上面创建的StorageClass，只不过这个时候我们需要手动来创建一个PVC

##### 创建PVC

```bash
# 新建ceph-storageclass-pvc.yml
vim ceph-storageclass-pvc.yml

kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: test-pvc
  annotations:
    volume.beta.kubernetes.io/storage-class: ceph-storageclass
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 50Gi

# 创建PVC
kubectl create -f ceph-storageclass-pvc.yml
```

##### 创建deployment

```bash
# 新建ceph-storageclass-nginx-deployment.yml
vim ceph-storageclass-nginx-deployment.yml

apiVersion: apps/v1beta1
kind: Deployment
metadata:
  name: ceph-nginx
spec:
  replicas: 3
  template:
    metadata:
      labels:
        app: ceph-nginx
    spec:
      containers:
      - name: ceph-nginx
        image: nginx
        ports:
        - containerPort: 80
        volumeMounts:
          - mountPath: "/data"
            name: data
      volumes:
        - name: data
          persistentVolumeClaim:
            claimName: test-pvc

# 创建deployment
kubectl create -f ceph-storageclass-nginx-deployment.yml

# 查看pods
kubectl get pods
NAME                          READY     STATUS    RESTARTS   AGE
ceph-nginx-3206996150-29q7j   1/1       Running   0          7m
ceph-nginx-3206996150-94tzk   1/1       Running   0          7m
ceph-nginx-3206996150-xvkzh   1/1       Running   0          7m

# 查看PV和PVC
kubectl get pvc
NAME                             STATUS    VOLUME                                     CAPACITY   ACCESSMODES   STORAGECLASS        AGE
test-pvc                         Bound     pvc-76a48238-9dcf-11e7-84a9-00155d201312   50Gi       RWX           ceph-storageclass   9m

kubectl get pv
NAME                                       CAPACITY   ACCESSMODES   RECLAIMPOLICY   STATUS    CLAIM                                    STORAGECLASS        REASON    AGE
pvc-76a48238-9dcf-11e7-84a9-00155d201312   50Gi       RWX           Delete          Bound     default/test-pvc                         ceph-storageclass             10m

# 进入pod 查看使用情况，看到/data总共50G
kubectl exec -ti ceph-nginx-3206996150-29q7j -- df -Th
Filesystem                                                                                      Type   Size  Used Avail Use% Mounted on
/dev/rbd0                                                                                       ext4    50G   52M   47G   1% /data
```

至此kubernetes结合ceph RBD的实验基本上已经完成，我们发现，storageclass确实是个好东西，省去了创建PV的步骤，并且，可以根据PVC中定义的class来选择创建不同的PVC
