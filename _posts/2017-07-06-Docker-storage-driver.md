---
layout: post
title: Docker基础-docker 存储驱动
categories: [docker]
description: docker storage
keywords: docker
---
# Select a storage driver

该篇主要介绍Docker的存储驱动，列出了Docker支持的存储驱动以及介绍了用来管理这些存储驱动的命令，同时也提供了在为Docker选择存储驱动的一些建议。
<!--more-->
# 热插拔存储驱动架构

Docker提供了一个可热插拔的存储驱动架构，这意味着，你可以灵活的插入最合适你当前环境的存储驱动程序。每一个Docker存储驱动都是基于Linux文件系统或卷组管理器。
每个存储驱动有自己独特的方式来管理数据层和镜像层，这意味着特定的驱动程序在特定的环境下，运行的会比其他的存储驱动好。

一旦你决定使用那个存储驱动，那么在启动docker的时候，你就指定该驱动程序，因此，Docker每次只能运行一个存储驱动，并且所有的容器实例都是用该存储驱动。下表列出了所有支持的存储驱动：

| Technology    | Storage driver name  |
| ----------    | :-------------------:|
| [OverlayFS](https://docs.docker.com/engine/userguide/storagedriver/overlayfs-driver/)     | overlay or overlay2  |
| [AUFS](https://docs.docker.com/engine/userguide/storagedriver/aufs-driver/)          | aufs                 |
| [Btrfs](https://docs.docker.com/engine/userguide/storagedriver/btrfs-driver/)         | btrfs                |
| [Device Mapper](https://docs.docker.com/engine/userguide/storagedriver/device-mapper-driver/) | devicemapper         |
| VFS           | vfs                  |
| [ZFS](https://docs.docker.com/engine/userguide/storagedriver/zfs-driver/)           | zfs                  |
到底使用哪个存储驱动，部分取决于你本地后台的文件系统，比如说，你当前主机的文件系统是btrfs，那么你的存储驱动最好也选择btrfs，下表列出了每个存储驱动和它对应所匹配的后台文件系统：

| Storage driver | Commonly used on | Disabled on |
| -------------- | :---------------- | :----------- |
| overlay        | ext4 xfs         | btrfs aufs overlay overlay2 zfs eCryptfs |
| overlay2       | ext4 xfs         | btrfs aufs overlay overlay2 zfs eCryptfs |
| aufs           | ext4 xfs         | btrfs aufs eCryptfs |
| devicemapper   | direct-lvm       | N/A |
| btrfs | btrfs *only* | N/A |
| vfs | debugging *only* | N/A |
| zfs | zfs *only* | N/A |

查看当前Docker使用的存储驱动命令
```bash
$ docker info
Containers: 10
 Running: 1
 Paused: 0
 Stopped: 9
Images: 5
Server Version: 1.12.3
Storage Driver: devicemapper
Backing Filesystem: xfs
```
在启动时Docker时指定存储驱动
```bash
$ dockerd --storage-driver=overlay
```

# 共享存储系统和存储驱动
许多企业使用共享存储（如SAN和NAS等），而Docker存储驱动程序是可以在这些共享存储上操作的，这是Docker在存储上有了更高的性能和可用性，但是目前，Docker还没有与这些共享存储进行集成，虽然没有和共享存储集成，但是，Docker的存储驱动程序是基于Linux文件系统和卷管理器的，所以Docker也能很好的在共享存储上进行使用。

# 选择存储驱动建议
在选择存储驱动之前，你必须了解到以下两点：
1.没有那个存储驱动最适合于那个用例
2.存储驱动正序正在不停的发展和改进
所以，我们在选择存储驱动的时候要注意：
1.稳定性
> * 使用默认的存储驱动
> * 使用Docker指定的稳定版的存储驱动

2.是否有使用该存储驱动的经验
你的开发团队是否有使用该存储驱动的经验，这也是你在选择存储驱动的时候需要考虑的

3.该存储驱动是否发展成熟
许多人认为`OverlayFS`是Docker 存储驱动未来发展的趋势，但是，相对于`aufs`和`devicemapper`来说，它还不太成熟，所以在选择存储驱动的时候，要尽可能的选择已经发展成熟的存储驱动
