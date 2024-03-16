---
title: "镜像文件系统：配置 Kubernetes 将容器存储在单独的文件系统上"
description: "Image Filesystem: Configuring Kubernetes to store containers on a separate filesystem"
date: 2024-03-06T00:08:00+08:00
image: https://img.picgo.net/2024/03/05/_20240305223322f97c305c6e79f24e.jpeg
tags: [ "Kubernetes", "Kubernetes blog" ]
categories: [ "Kubernetes" ]
slug: kubernetes-separate-image-filesystem
---

<!--
layout: blog
title: 'Image Filesystem: Configuring Kubernetes to store containers on a separate filesystem'
date: 2024-01-23
slug: kubernetes-separate-image-filesystem
-->

<!--
**Author:** Kevin Hannon (Red Hat)
-->

**作者：** Kevin Hannon (Red Hat)

**译者：** [Rui Yang](https://github.com/techoc)

<!-- 
A common issue in running/operating Kubernetes clusters is running out of disk space.
When the node is provisioned, you should aim to have a good amount of storage space for your container images and running containers. 
-->

运行或者操作 Kubernetes 集群的一个常见问题是磁盘空间不足。
在预分配节点时，你的目标应该是为容器镜像和正在运行的容器提供大量存储空间。

<!-- 
The [container runtime](/docs/setup/production-environment/container-runtimes/) usually writes to `/var`.
This can be located as a separate partition or on the root filesystem.
CRI-O, by default, writes its containers and images to `/var/lib/containers`, while containerd writes its containers and images to `/var/lib/containerd`. 
-->

[容器运行时](https://kubernetes.io/docs/setup/production-environment/container-runtimes/)通常会写入到 `/var`目录下。
他可以位于单独的分区或者位于根文件系统上。
在默认情况下，CRI-O 将其容器和镜像写入 `/var/lib/containers`目录下，而 containerd 将其容器和镜像写入 `/var/lib/containerd`
目录下。

<!-- 
In this blog post, we want to bring attention to ways that you can configure your container runtime to store its content separately from the default partition.
This allows for more flexibility in configuring Kubernetes and provides support for adding a larger disk for the container storage while keeping the default filesystem untouched. 
-->

在这篇博文中，我们希望引起您的注意，您可以配置容器运行时以将其内容与默认分区分开存储。
这使得配置 Kubernetes 变得更加灵活，并支持为容器存储添加更大的磁盘，同时保持默认文件系统不变。

<!-- 
One area that needs more explaining is where/what Kubernetes is writing to disk. 
-->

需要详细了解的是 Kubernetes 会写入到磁盘哪里以及写入哪些内容。

<!-- 
## Understanding Kubernetes disk usage

Kubernetes has persistent data and ephemeral data. The base path for the kubelet and local
Kubernetes-specific storage is configurable, but it is usually assumed to be `/var/lib/kubelet`.
In the Kubernetes docs, this is sometimes referred to as the root or node filesystem. The bulk of this data can be categorized into:

- ephemeral storage
- logs
- and container runtime

This is different from most POSIX systems as the root/node filesystem is not `/` but the disk that `/var/lib/kubelet` is on. 
-->

## 理解 Kubernetes 磁盘的使用

Kubernetes 有持久化的数据和临时数据。kubelet 和本地 Kubernetes
特定的存储的根路径是可配置的，但是通常认为它位于 `/var/lib/kubelet`。在 Kubernetes 文档中，这有时被称为根或节点文件系统。该数据的大部分可以分为：

- 临时存储
- 日志
- 容器运行时

这与大多数 POSIX 系统不同，因为根或者节点文件系统不是 `/`，而是 `/var/lib/kubelet` 所在的磁盘。

<!-- 
### Ephemeral storage

Pods and containers can require temporary or transient local storage for their operation.
The lifetime of the ephemeral storage does not extend beyond the life of the individual pod, and the ephemeral storage cannot be shared across pods. 
-->

### 临时存储

Pod 和容器可能需要临时或暂时的本地存储来进行操作。
临时存储的生命周期不会超过单个 pod 的生命周期，并且临时存储不能跨 pod 共享。

<!-- 
### Logs

By default, Kubernetes stores the logs of each running container, as files within `/var/log`.
These logs are ephemeral and are monitored by the kubelet to make sure that they do not grow too large while the pods are running. 
-->

### 日志

默认情况下，Kubernetes 会将每个运行容器的日志，作为文件存储在 `/var/log` 目录下。
这些日志是临时的，由 kubelet 监控，以确保它们在运行期间不会变得太大。

<!-- 
You can customize the [log rotation](/docs/concepts/cluster-administration/logging/#log-rotation) settings
for each node to manage the size of these logs, and configure log shipping (using a 3rd party solution)
to avoid relying on the node-local storage. 
-->

你可以自定义每个节点的 [日志轮转](https://kubernetes.io/docs/concepts/cluster-administration/logging/#log-rotation)
设置，以管理这些日志的大小，以及配置日志传输（使用第三方解决方案）以避免依赖于节点本地存储。

<!-- 
### Container runtime

The container runtime has two different areas of storage for containers and images. 
-->

### 容器运行时

容器运行时有两个不同的存储区域用于容器和镜像。

<!--
- read-only layer: Images are usually denoted as the read-only layer, as they are not modified when containers are running.
  The read-only layer can consist of multiple layers that are combined into a single read-only layer.
  There is a thin layer on top of containers that provides ephemeral storage for containers if the container is writing to the filesystem.
-->

- 只读层: 镜像通常被标记为只读层，因为它们在容器运行时不会被修改。
  只读层可以由多个层组成，然后组合成一个单独的只读层。
  在容器的顶层上，有一个薄层提供容器的临时存储，如果容器在文件系统上写入，则其可以提供为临时存储。

<!--
- writeable layer: Depending on your container runtime, local writes might be
  implemented as a layered write mechanism (for example, `overlayfs` on Linux or CimFS on Windows).
  This is referred to as the writable layer.
  Local writes could also use a writeable filesystem that is initialized with a full clone of the container
  image; this is used for some runtimes based on hypervisor virtualisation.
  -->

- 可写层: 依赖于您的容器运行时，本地写入可能会使用基于层的写机制（例如，Linux 上的 `overlayfs` 或 Windows 上的 `CimFS` ）。
  这被称为可写层。
  本地写入也可以使用可写文件系统，该文件系统使用容器镜像的完整克隆进行初始化;这用于基于虚拟机管理程序的一些运行时。

<!--
The container runtime filesystem contains both the read-only layer and the writeable layer.
This is considered the `imagefs` in Kubernetes documentation.
-->

容器运行时文件系统包含只读层和可写层。
在 Kubernetes 文档中，这被称为 `imagefs`。

<!--
## Container runtime configurations

### CRI-O
-->

## 容器运行时配置

### CRI-O

<!--
CRI-O uses a storage configuration file in TOML format that lets you control how the container runtime stores persistent and temporary data.
CRI-O utilizes the [storage library](https://github.com/containers/storage).
Some Linux distributions have a manual entry for storage (`man 5 containers-storage.conf`).
The main configuration for storage is located in `/etc/containers/storage.conf` and one can control the location for temporary data and the root directory.
The root directory is where CRI-O stores the persistent data.
-->

CRI-O 使用 TOML 格式的存储配置文件，让你可以控制容器运行时存储持久化和临时数据。
CRI-O 使用 [storage library](https://github.com/containers/storage)。
一些 Linux 发行版提供了存储的手册页 (`man 5 containers-storage.conf`)。
存储的主要配置位于 `/etc/containers/storage.conf` ，用户可以控制临时数据的位置和根目录。
根目录是 CRI-O 存储持久数据的位置。

<!--
```toml
[storage]
# Default storage driver
driver = "overlay"
# Temporary storage location
runroot = "/var/run/containers/storage"
# Primary read/write location of container storage
graphroot = "/var/lib/containers/storage"
```
-->

```toml
[storage]
# 默认存储驱动程序
driver = "overlay"
# 临时存储位置
runroot = "/var/run/containers/storage"
# 容器存储的主要读/写位置
graphroot = "/var/lib/containers/storage"
```

<!--
- `graphroot`
  - Persistent data stored from the container runtime
  - If SELinux is enabled, this must match the `/var/lib/containers/storage`
- `runroot`
  - Temporary read/write access for container
  - Recommended to have this on a temporary filesystem
-->

- `graphroot`
    - 容器运行时的持久数据存储
    - 如果启用 SELinux，则必须与 `/var/lib/containers/storage` 相匹配
- `runroot`
    - 临时访问容器的读/写访问
    - 建议将此放在临时文件系统中

<!--
Here is a quick way to relabel your graphroot directory to match `/var/lib/containers/storage`:

```bash
semanage fcontext -a -e /var/lib/containers/storage <YOUR-STORAGE-PATH>
restorecon -R -v <YOUR-STORAGE-PATH>
```
-->

这里有一个快速的方法来重新标记你的 graphroot 目录，以匹配`/var/lib/containers/storage`:

```bash
semanage fcontext -a -e /var/lib/containers/storage <YOUR-STORAGE-PATH>
restorecon -R -v <YOUR-STORAGE-PATH>
```

<!--
### containerd

The containerd runtime uses a TOML configuration file to control where persistent and ephemeral data is stored.
The default path for the config file is located at `/etc/containerd/config.toml`.

The relevant fields for containerd storage are `root` and `state`.
-->

### containerd

containerd 使用 TOML 配置文件来控制持久数据和临时数据的存储位置。
默认的配置文件路径位于 `/etc/containerd/config.toml`。

containerd 存储的相关字段是 `root` 和 `state`。

<!--
- `root`
  - The root directory for containerd metadata
  - Default is `/var/lib/containerd`
  - Root also requires SELinux labels if your OS requires it
- `state`
  - Temporary data for containerd
  - Default is `/run/containerd`
-->

- `root`
    - 容器运行时元数据的根目录
    - 默认位置为 `/var/lib/containerd`
    - 如果您的操作系统需要，Root 也需要 SELinux 标签
- `state`
    - 容器运行时的临时数据
    - 默认位置为 `/run/containerd`

<!--
## Kubernetes node pressure eviction
-->

## Kubernetes 节点压力驱逐

<!--
Kubernetes will automatically detect if the container filesystem is split from the node filesystem.
When one separates the filesystem, Kubernetes is responsible for monitoring both the node filesystem and the container runtime filesystem.
Kubernetes documentation refers to the node filesystem and the container runtime filesystem as nodefs and imagefs.
If either nodefs or the imagefs are running out of disk space, then the overall node is considered to have disk pressure.
Kubernetes will first reclaim space by deleting unusued containers and images, and then it will resort to evicting pods.
On a node that has a nodefs and an imagefs, the kubelet will
[garbage collect](/docs/concepts/architecture/garbage-collection/#containers-images) unused container images
on imagefs and will remove dead pods and their containers from the nodefs.
If there is only a nodefs, then Kubernetes garbage collection includes dead containers, dead pods and unused images.
-->

Kubernetes 将自动检测容器文件系统是否与节点文件系统分离。
当将文件系统分离时，Kubernetes 负责监控节点文件系统和容器运行时文件系统。
Kubernetes 文档将节点文件系统和容器运行时文件系统称为 nodefs 和 imagefs。
如果 nodefs 或 imagefs 中的任何一个磁盘空间不足，则整个节点被认为存在磁盘压力。
Kubernetes 首先通过删除未使用的容器和镜像来回收空间，然后将逐出 Pod。
在具有 nodefs 和 imagefs 的节点上，kubelet 将在 imagefs 上使用[垃圾收集](https://kubernetes.io/zh-cn/docs/concepts/architecture/garbage-collection/#containers-images)来清理未使用的容器镜像，并从 nodefs 中删除死亡的 Pod 及其容器。
如果只有一个 nodefs，则 Kubernetes 垃圾收集包括死亡的容器、死亡的 Pod 和未使用的镜像。

<!--
Kubernetes allows more configurations for determining if your disk is full.
The eviction manager within the kubelet has some configuration settings that let you control
the relevant thresholds.
For filesystems, the relevant measurements are `nodefs.available`, `nodefs.inodesfree`, `imagefs.available`, and `imagefs.inodesfree`.
If there is not a dedicated disk for the container runtime then imagefs is ignored.
-->

Kubernetes 提供了许多配置选项来确定磁盘是否已满。
kubelet 中的驱逐管理器有一些配置设置，可以让你控制相关的阈值。
对于文件系统，相关的度量是`nodefs.available`、`nodefs.inodesfree`、`imagefs.available`和`imagefs.inodesfree`。
如果没有专门用于容器运行时的磁盘，则会忽略 imagefs。

<!--
Users can use the existing defaults:

- `memory.available` < 100MiB
- `nodefs.available` < 10%
- `imagefs.available` < 15%
- `nodefs.inodesFree` < 5% (Linux nodes)

Kubernetes allows you to set user defined values in `EvictionHard` and `EvictionSoft` in the kubelet configuration file.
-->

用户可以使用现有的默认值有：

- `memory.available` < 100MiB
- `nodefs.available` < 10%
- `imagefs.available` < 15%
- `nodefs.inodesFree` < 5% (Linux 节点)

Kubernetes 允许您在 kubelet 配置文件中设置用户自定义值，具体位置为 `EvictionHard` 和 `EvictionSoft` 配置项。

<!--
`EvictionHard`
: defines limits; once these limits are exceeded, pods will be evicted without any grace period.

`EvictionSoft`
: defines limits; once these limits are exceeded, pods will be evicted with a grace period that can be set per signal.
-->

`EvictionHard`
: 定义限制；一旦超过这些限制，Pod 将立即驱逐，没有任何宽限期。

`EvictionSoft`
: 定义限制；一旦超过这些限制，Pod 将按照各信号所设置的宽限期后被。

<!--
If you specify a value for `EvictionHard`, it will replace the defaults.
This means it is important to set all signals in your configuration.

For example, the following kubelet configuration could be used to configure [eviction signals](/docs/concepts/scheduling-eviction/node-pressure-eviction/#eviction-signals-and-thresholds) and grace period options.
-->

如果你为 `EvictionHard` 指定了值，那么它将替代默认值。
这意味着在你的配置中设置所有的信号显得非常重要。

举个例子，以下的 kubelet 配置可用于配置[驱逐信号](https://kubernetes.io/zh-cn/docs/concepts/scheduling-eviction/node-pressure-eviction/#eviction-signals-and-thresholds)和宽限期选项。

```yaml
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
address: "192.168.0.8"
port: 20250
serializeImagePulls: false
evictionHard:
  memory.available: "100Mi"
  nodefs.available: "10%"
  nodefs.inodesFree: "5%"
  imagefs.available: "15%"
  imagefs.inodesFree: "5%"
evictionSoft:
  memory.available: "100Mi"
  nodefs.available: "10%"
  nodefs.inodesFree: "5%"
  imagefs.available: "15%"
  imagefs.inodesFree: "5%"
evictionSoftGracePeriod:
  memory.available: "1m30s"
  nodefs.available: "2m"
  nodefs.inodesFree: "2m"
  imagefs.available: "2m"
  imagefs.inodesFree: "2m"
evictionMaxPodGracePeriod: 60s
```

<!--
### Problems

The Kubernetes project recommends that you either use the default settings for eviction or you set all the fields for eviction.
You can use the default settings or specify your own `evictionHard` settings. If you miss a signal, then Kubernetes will not monitor that resource.
One common misconfiguration administrators or users can hit is mounting a new filesystem to `/var/lib/containers/storage` or `/var/lib/containerd`.
Kubernetes will detect a separate filesystem, so you want to make sure to check that `imagefs.inodesfree` and `imagefs.available` match your needs if you've done this.
-->

### 问题

Kubernetes 项目建议你遵循默认驱逐设置或着设置所有驱逐字段。
你可以使用默认设置或指定你自己的 `evictionHard` 设置。 如果你漏掉一个信号，那么 Kubernetes 将不会监控该资源。
管理员或用户常见的一个配置错误是将新的文件系统挂载在 `/var/lib/containers/storage` 或 `/var/lib/containerd` 目录上。Kubernetes 会检测到一个单独的文件系统，因此如果你已经这样做，请确保检查 `imagefs.inodesfree` 和 `imagefs.available` 符合你的需要。

<!--
Another area of confusion is that ephemeral storage reporting does not change if you define an image
filesystem for your node. The image filesystem (`imagefs`) is used to store container image layers; if a
container writes to its own root filesystem, that local write doesn't count towards the size of the container image. The place where the container runtime stores those local modifications is runtime-defined, but is often
the image filesystem.
If a container in a pod is writing to a filesystem-backed `emptyDir` volume, then this uses space from the
`nodefs` filesystem.
The kubelet always reports ephemeral storage capacity and allocations based on the filesystem represented
by `nodefs`; this can be confusing when ephemeral writes are actually going to the image filesystem.
-->

另一个令人困惑的是，当为节点定义一个镜像文件系统时，临时存储报告不会改变。镜像文件系统(`imagefs`)被用于存储容器镜像层；如果一个容器向自己的根文件系统写入，那么这种本地写入不会计入到容器镜像的大小中。容器运行时用于存储本地修改的位置由运行时本身决定，通常情况下，这个位置就是镜像文件系统。
如果一个 Pod 中的容器在 `emptyDir` 文件系统上写入数据，那么它将使用 `nodefs` 文件系统中的空间。
kubelet 一直根据 `nodefs` 表示的文件系统报告临时存储容量和分配情况;当临时写入操作实际上是写到镜像文件系统时，这种差别可能会让人困惑

<!--
### Future work

To fix the ephemeral storage reporting limitations and provide more configuration options to the container runtime, SIG Node are working on [KEP-4191](http://kep.k8s.io/4191).
In KEP-4191, Kubernetes will detect if the writeable layer is separated from the read-only layer (images).
This would allow us to have all ephemeral storage, including the writeable layer, on the same disk as well as allowing for a separate disk for images.
-->

### 后续工作

为了修复临时存储报告的相关限制和为容器运行时提供更多的配置选项，SIG Node 正在开发 [KEP-4191](http://kep.k8s.io/4191)。
在 KEP-4191 中，Kubernetes 将检测可写层和只读层（镜像）是否分离。
这可以让我们将所有临时存储，包括可写层，都放在同一个磁盘上，同时还可以为镜像分配一个单独的磁盘。

<!--
### Getting involved

If you would like to get involved, you can
join [Kubernetes Node Special-Interest-Group](https://github.com/kubernetes/community/tree/master/sig-node) (SIG).
-->

### 参与其中

如果你有兴趣参与其中，你可以加入 [Kubernetes Node 特性兴趣小组](https://github.com/kubernetes/community/tree/master/sig-node) (SIG)。

<!--
If you would like to share feedback, you can do so on our
[#sig-node](https://kubernetes.slack.com/archives/C0BP8PW9G) Slack channel.
If you're not already part of that Slack workspace, you can visit https://slack.k8s.io/ for an invitation.

Special thanks to all the contributors who provided great reviews, shared valuable insights or suggested the topic idea.
-->

如果你有兴趣分享反馈，你可以分享到我们的[#sig-node](https://kubernetes.slack.com/archives/C0BP8PW9G) Slack 频道。
如果你还没有加入该 Slack 工作空间，请访问 https://slack.k8s.io/ 获取邀请。

特别感谢所有提供出色评审、分享宝贵见解或建议主题想法的贡献者。

- Peter Hunt
- Mrunal Patel
- Ryan Phillips
- Gaurav Singh

> 原文地址: https://kubernetes.io/blog/2024/01/23/kubernetes-separate-image-filesystem/
> 
> 官方译文: https://kubernetes.io/zh-cn/blog/2024/01/23/kubernetes-separate-image-filesystem/