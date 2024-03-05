---
title: "Image Filesystem: Configuring Kubernetes to store containers on a separate filesystem"
description: 
date: 2024-03-06T00:08:00+08:00
image: https://img.picgo.net/2024/03/05/_20240305223322f97c305c6e79f24e.jpeg
hidden: false
comments: true
draft: false
tags:
  - Kubernetes
categories: ["Kubernetes"]
---

作者： Kevin Hannon（红帽）

运行/操作库伯内特斯集群的一个常见问题是磁盘空间不足。当配置节点时，您应该致力于为您的容器映像和正在运行的容器提供大量存储空间。容器运行时通常写入 `/var`.这可以作为单独的分区或位于根文件系统上。默认情况下，`CRI-O` 将其容器和映像写入 `/var/lib/containers`，而容器将其容器和映像写入 `/var/lib/containerd`.

在这篇博文中，我们希望关注如何配置容器运行时以将其内容与默认分区分开存储。
这使得配置库伯内特斯更加灵活，并支持为容器存储添加更大的磁盘，同时保持默认文件系统不变。

需要更多解释的一个领域是库伯内特斯在哪里/写什么到磁盘。