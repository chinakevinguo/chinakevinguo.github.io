---
layout: post
title: Docker基础-docker limit资源限制
categories: [docker]
description: docker limit资源限制
keywords: docker
---
默认情况下，容器没有资源约束，可以使用与主机的内核调度程序允许的的资源一样多的资源。docker提供了在运行`docker run`指定选项来控制容器内存，cpu或IO的方法。这部分提供了有关何时应该设置这类限制以及设置这些限制可能有什么影响的详细信息。

<!--more-->
# Memory
Docker 可以强制执行硬内存限制，其允许容器使用不超过用户或系统给定的内存大小，或者软限制，允许容器在满足条件的情况下使用所需内存，例如,当内核检测到主机上的内存不足或有争抢。下面的选项单独使用或集中使用会有不同的效果。
这些选项中大部分都是采用正整数，后面跟一个后缀b,k,m,g，表示字节，千字节，兆字节，千兆字节。

| Option | Description |
| :---- | :----------- |
| -m or --memory= | 容器可以使用的最大内存大小，如果你设置了这个选项，那么内存最小也要4m |
| --memory-swarp* | 容器可以交换到硬盘的内存大小，可参考[--memory-swap details](https://docs.docker.com/engine/admin/resource_constraints/#memory-swap-details) |
| --memory-swappiness | 默认情况下，主机内核可以交换容器使用的匿名页面的百分比，可以在0-100之间设置`--memory-swappiness`的百分比，可参考[--memory-swappiness details](https://docs.docker.com/engine/admin/resource_constraints/#memory-swappiness-details) |
| --memory-reservation | 允许你指定一个小于`--memory`的软限制，当docker在主机上检测到争用或低内存时激活该限制。如果使用`--memory-reservation`，则必须将其设置为低于`--memory`，以使其优先。因为它是一个软限制，不保证容器不会超过限制。 |
| --kernel-memory | 容器可以使用的最大内核内存，最小值是4m。因为内核内存无法交换出来，一个缺少内核内存的容器可能会阻塞主机资源，这会对主机和其他容器产生副作用 |
| --oom-kill-disable | 默认情况下，如果发生内存不足(OOM)错误，内核会杀死容器中的进程。我们可以使用`--oom-kill-disable`选项来改变设置，如果你设置了`-m/--memory`选项，那么容器会一直耗尽到`-m`限制的内存大小。而如果你没有设定`-m`，主机会尽可能的耗尽内存，内核可能需要杀死主机系统的进程以释放内存。

## --memory-swap details

> * 如果没有设置`--memory-swap`，而设置了`--memory`，容器能够使用`--memory`值的两倍的swap。例如：`--memory=300m`,`--memory-swap`没有设置，那么容器能够使用300m内存和600m swap。
> * 如果`--memory`和`--memory-swap`都设置了，`--memory-swap`表示能够使用内存和swap的总数，`--memory`控制非swap的内存大小。例如：`--memory=300m`,`--memory-swap=1g`，那么容器能够使用300m内存和700m的swap。
> * 如果设置为-1(默认)，表示容器可以无限使用swap

## --memory-swappiness details

> * 如果值为0，关闭匿名页面交换
> * 值100，将所有匿名页面设置为可交换
> * 默认情况下，如果不设置`--memory-swappiness`，容器将继承主机计算机的值

## --kernel-memory details

内核内存限制以分配给容器的总内存来表示，有以下几种场景：

> * 没有限制`--memory`，没有限制`--kernel-memory`： 这是默认值
> * 没有限制`--memory`，有限制`--kernel-memory`: 当所有的cgroups所需的内存量超过主机实际拥有的内存量时，适合设置为这样子。你可以设置内核内存不超过主机可用内存，容器需要更多内存只能等待了。
> * 有限制`--memory`，没有限制`--kernel-memory`：总内存是限制的，不过内核内存无限制
> * 有限制`--memory`，有限制`--kernel-memory`：用户和内核限制都限制时，对调试内存相关问题会有帮助。如果一个容器用完了这两种内存之中的一种，它不会影响到其他的容器和主机。如果内核内存限制比用户内存低，使用完内核内存后，会导致容器长生OOM错误，反之，则不会。

# CPU

默认情况下，所有容器获得CPU周期的比例相同，你可以通过下面的选项来对容器的CPU使用进行调整。

| Option | Description |
| :----- | :--------- |
| --cpu-shares | 设置权重，修改默认的1024增大或者减小。当其他容器有空闲CPU时，其他容器可使用空闲CPU时间 |
| --cpu-period | 容器的一个逻辑CPU的调度周期。默认值是100000（100ms），当然我们也可以自己设置CPU周期，限制容器CPU用量，通常和`--cpu-quota`参数使用 |
| --cpu-quota | 在由`--cpu-period`设置的时间段内，容器可以调度的最大CPU使用量，默认是0，以为着允许容器获得1个CPU的100%的资源量。设置50000限制CPU资源的50% |
| --cpuset-cpus | 定容器允许运行的CPU号(在多核心系统中) |

# Block IO

有两个选项可用于调整容器对直连块IO设备的访问。你还可以按照每秒字节数或每秒IO操作来指定带宽限制。

| Option | Description |
| :----- | :--------- |
| blkio-weight | 默认情况下，每个容器可以使用相同比例的IO带宽。默认权重是500，要提高或者降低，可以设置`--blkio-weight`来设置介于10-1000之间的值，此设置会平等的影响到所有块IO设备 |
| blkio-weight-device | 与`--blkio-weight`相同，但你可以使用`--blkio-weight-device=DEVICE_NAME:WEGITH`为每个设备设置权重。 |
| --device-read-bps和--device-write-bps | 根据大小限制设备读取或写入的速率，使用kb，mb或gb后缀 |
| --device-read-iops和--device-write-iops | 通过IO操作/秒限制设备读取或写入的速率 |
