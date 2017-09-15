---
layout: post
title: kubernetes ceph 笔记 2
categories: [kubernetes,ceph]
description: kubernetes ceph
keywords: kubernetes,ceph
---

> 其实我们在上一章的时候，已经简单的介绍了ceph中的基础组件以及其他软体架构，那么这一章，主要是来详细的介绍下ceph的工作原理、流程以及更实际的操作，比如ceph 内部的CRUSH bucket调整，PG/PGP参数调整等。并在最后简单的介绍了些CEPH集群硬件要求等。

### 寻址流程

通过对Ceph系统中寻址流程的了解来熟悉Ceph中的几个概念，寻址流程如下图

![寻址流程](/images/posts/Addressing.png)
