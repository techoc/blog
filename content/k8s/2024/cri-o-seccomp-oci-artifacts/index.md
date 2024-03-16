---
title: "CRI-O：应用来自OCI注册中心的seccomp配置文件"
description: "CRI-O: Applying seccomp profiles from OCI registries"
date: 2024-03-08T00:23:10+08:00
image: https://s.cn.bing.net/th?id=OHR.IguazuFalls_ZH-CN4749837052_1920x1080.webp&qlt=50
tags: ["Kubernetes", "Kubernetes blog"]
categories: ["Kubernetes"]
slug: cri-o-seccomp-oci-artifacts
---

<!--

---
layout: blog
title: "CRI-O: Applying seccomp profiles from OCI registries"
date: 2024-03-07
slug: cri-o-seccomp-oci-artifacts
---

-->

<!--
**Author:** Sascha Grunert
-->

**作者:** Sascha Grunert

**译者:** [Rui Yang](https://github.com/techoc)

<!--
Seccomp stands for secure computing mode and has been a feature of the Linux
kernel since version 2.6.12. It can be used to sandbox the privileges of a
process, restricting the calls it is able to make from userspace into the
kernel. Kubernetes lets you automatically apply seccomp profiles loaded onto a
node to your Pods and containers.
-->

Seccomp 代表安全计算模式，自 Linux 内核 2.6.12 版本起成为其特性。它可以用于沙箱化进程的特权，限制其从用户空间向内核发出的调用。Kubernetes 允许你自动应用加载到节点上的 seccomp 配置文件到你的 Pods 和容器。

<!--
But distributing those seccomp profiles is a major challenge in Kubernetes,
because the JSON files have to be available on all nodes where a workload can
possibly run. Projects like the [Security Profiles
Operator](https://sigs.k8s.io/security-profiles-operator) solve that problem by
running as a daemon within the cluster, which makes me wonder which part of that
distribution could be done by the [container
runtime](/docs/setup/production-environment/container-runtimes).

Runtimes usually apply the profiles from a local path, for example:
-->

但是在 Kubernetes 中分发这些 seccomp 配置文件是一个重大的挑战，因为 JSON 文件必须在所有可能运行工作负载的节点上可用。像 [Security Profiles](https://sigs.k8s.io/security-profiles-operator) 这样的项目通过在集群内运行守护进程来解决这个问题，这让我想知道哪些部分可以由 [容器运行时](/docs/setup/production-environment/container-runtimes) 来完成。

运行时通常从本地路径应用配置文件，例如：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod
spec:
  containers:
    - name: container
      image: nginx:1.25.3
      securityContext:
        seccompProfile:
          type: Localhost
          localhostProfile: nginx-1.25.3.json
```

<!--
The profile `nginx-1.25.3.json` has to be available in the root directory of the
kubelet, appended by the `seccomp` directory. This means the default location
for the profile on-disk would be `/var/lib/kubelet/seccomp/nginx-1.25.3.json`.
If the profile is not available, then runtimes will fail on container creation
like this:
-->

配置文件 `nginx-1.25.3.json` 必须位于 kubelet 的根目录中，并追加 `seccomp` 目录。这意味着配置文件在磁盘上的默认位置将是 `/var/lib/kubelet/seccomp/nginx-1.25.3.json`。如果配置文件不可用，那么容器创建时容器运行时会失败，如下所示：

```shell
kubectl get pods
```

```console
NAME   READY   STATUS                 RESTARTS   AGE
pod    0/1     CreateContainerError   0          38s
```

```shell
kubectl describe pod/pod | tail
```

```console
Tolerations:                 node.kubernetes.io/not-ready:NoExecute op=Exists for 300s
                             node.kubernetes.io/unreachable:NoExecute op=Exists for 300s
Events:
  Type     Reason     Age                 From               Message
  ----     ------     ----                ----               -------
  Normal   Scheduled  117s                default-scheduler  Successfully assigned default/pod to 127.0.0.1
  Normal   Pulling    117s                kubelet            Pulling image "nginx:1.25.3"
  Normal   Pulled     111s                kubelet            Successfully pulled image "nginx:1.25.3" in 5.948s (5.948s including waiting)
  Warning  Failed     7s (x10 over 111s)  kubelet            Error: setup seccomp: unable to load local profile "/var/lib/kubelet/seccomp/nginx-1.25.3.json": open /var/lib/kubelet/seccomp/nginx-1.25.3.json: no such file or directory
  Normal   Pulled     7s (x9 over 111s)   kubelet            Container image "nginx:1.25.3" already present on machine
```

<!--
The major obstacle of having to manually distribute the `Localhost` profiles
will lead many end-users to fall back to `RuntimeDefault` or even running their
workloads as `Unconfined` (with disabled seccomp).
-->

需要手动分发`Localhost`配置文件这个主要障碍将导致许多终端用户退回到`RuntimeDefault`，甚至运行其工作负载为`Unconfined`(禁用 seccomp)。

<!--
## CRI-O to the rescue

The Kubernetes container runtime [CRI-O](https://github.com/cri-o/cri-o)
provides various features using custom annotations. The v1.30 release
[adds](https://github.com/cri-o/cri-o/pull/7719) support for a new set of
annotations called `seccomp-profile.kubernetes.cri-o.io/POD` and
`seccomp-profile.kubernetes.cri-o.io/<CONTAINER>`. Those annotations allow you
to specify:

- a seccomp profile for a specific container, when used as:
  `seccomp-profile.kubernetes.cri-o.io/<CONTAINER>` (example:
  `seccomp-profile.kubernetes.cri-o.io/webserver:
'registry.example/example/webserver:v1'`)
- a seccomp profile for every container within a pod, when used without the
  container name suffix but the reserved name `POD`:
  `seccomp-profile.kubernetes.cri-o.io/POD`
- a seccomp profile for a whole container image, if the image itself contains
  the annotation `seccomp-profile.kubernetes.cri-o.io/POD` or
  `seccomp-profile.kubernetes.cri-o.io/<CONTAINER>`.
-->

## CRI-O 来解忧

Kubernetes 容器运行时 [CRI-O](https://github.com/cri-o/cri-o) 通过自定义注释提供各种功能。v1.30 版本添加了对一组新注释的[支持](https://github.com/cri-o/cri-o/pull/7719)，称为 `seccomp-profile.kubernetes.cri-o.io/POD` 和 `seccomp-profile.kubernetes.cri-o.io/<CONTAINER>`。这些注释允许你指定：

- 一个特定容器的 seccomp 配置文件，当作为 `seccomp-profile.kubernetes.cri-o.io/<CONTAINER>` 使用时（例如：`seccomp-profile.kubernetes.cri-o.io/webserver: 'registry.example/example/webserver:v1'`）
- 一个 pod 中所有容器的 seccomp 配置文件，当不使用容器名称后缀但使用保留名称 `POD` 时（例如：`seccomp-profile.kubernetes.cri-o.io/POD`）
- 整个容器镜像的 seccomp 配置文件，如果镜像本身包含注释 `seccomp-profile.kubernetes.cri-o.io/POD` 或 `seccomp-profile.kubernetes.cri-o.io/<CONTAINER>`。

<!--
CRI-O will only respect the annotation if the runtime is configured to allow it,
as well as for workloads running as `Unconfined`. All other workloads will still
use the value from the `securityContext` with a higher priority.

The annotations alone will not help much with the distribution of the profiles,
but the way they can be referenced will! For example, you can now specify
seccomp profiles like regular container images by using OCI artifacts:
-->

CRI-O 只有在运行时配置为允许的情况下才会遵守该注释，以及以 `Unconfined` 方式运行工作负载。所有其他的工作负载仍将
使用具有更高优先级的`securityContext`中的值

仅仅使用注释并不能很好地解决配置文件的分发问题，但是它们可以被引用！例如，现在你可以像使用 OCI artifacts 一样指定 seccomp 配置文件：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod
  annotations:
    seccomp-profile.kubernetes.cri-o.io/POD: quay.io/crio/seccomp:v2
spec: …
```

<!--
The image `quay.io/crio/seccomp:v2` contains a `seccomp.json` file, which
contains the actual profile content. Tools like [ORAS](https://oras.land) or
[Skopeo](https://github.com/containers/skopeo) can be used to inspect the
contents of the image:
-->

镜像 `quay.io/crio/seccomp:v2` 包含一个`seccomp.json`,该文件包含实际的配置文件内容。可以使用像 [ORAS](https://oras.land) 或 [Skopeo](https://github.com/containers/skopeo) 这样的工具来检查镜像的内容：

```shell
oras pull quay.io/crio/seccomp:v2
```

```console
Downloading 92d8ebfa89aa seccomp.json
Downloaded  92d8ebfa89aa seccomp.json
Pulled [registry] quay.io/crio/seccomp:v2
Digest: sha256:f0205dac8a24394d9ddf4e48c7ac201ca7dcfea4c554f7ca27777a7f8c43ec1b
```

```shell
jq . seccomp.json | head
```

```yaml
{
  "defaultAction": "SCMP_ACT_ERRNO",
  "defaultErrnoRet": 38,
  "defaultErrno": "ENOSYS",
  "archMap": [
    {
      "architecture": "SCMP_ARCH_X86_64",
      "subArchitectures": [
        "SCMP_ARCH_X86",
        "SCMP_ARCH_X32"
```

```shell
# Inspect the plain manifest of the image
skopeo inspect --raw docker://quay.io/crio/seccomp:v2 | jq .
```

```yaml
{
  "schemaVersion": 2,
  "mediaType": "application/vnd.oci.image.manifest.v1+json",
  "config":
    {
      "mediaType": "application/vnd.cncf.seccomp-profile.config.v1+json",
      "digest": "sha256:ca3d163bab055381827226140568f3bef7eaac187cebd76878e0b63e9e442356",
      "size": 3,
    },
  "layers":
    [
      {
        "mediaType": "application/vnd.oci.image.layer.v1.tar",
        "digest": "sha256:92d8ebfa89aa6dd752c6443c27e412df1b568d62b4af129494d7364802b2d476",
        "size": 18853,
        "annotations": { "org.opencontainers.image.title": "seccomp.json" },
      },
    ],
  "annotations": { "org.opencontainers.image.created": "2024-02-26T09:03:30Z" },
}
```

<!--
The image manifest contains a reference to a specific required config media type
(`application/vnd.cncf.seccomp-profile.config.v1+json`) and a single layer
(`application/vnd.oci.image.layer.v1.tar`) pointing to the `seccomp.json` file.
But now, let's give that new feature a try!
-->

镜像清单包含对特定所需配置媒体类型的引用（`application/vnd. cncf.seccomp-profile.config.v1+json`）和指向`seccomp.json`文件的单个层（`application/vnd. oi.image.layer.v1.tar`）。但是现在，让我们试一试这个新特征！

<!--
### Using the annotation for a specific container or whole pod

CRI-O needs to be configured adequately before it can utilize the annotation. To
do this, add the annotation to the `allowed_annotations` array for the runtime.
This can be done by using a drop-in configuration
`/etc/crio/crio.conf.d/10-crun.conf` like this:
-->

### 使用注释来指定特定容器或整个 pod

CRI-O 需要适当地配置才能使用该注释。可以通过将注释添加到运行时的 `allowed_annotations` 数组来实现。可以使用一个 drop-in 配置文件 `/etc/crio/crio.conf.d/10-crun.conf` 来实现：

```toml
[crio.runtime]
default_runtime = "crun"

[crio.runtime.runtimes.crun]
allowed_annotations = [
    "seccomp-profile.kubernetes.cri-o.io",
]
```

<!--
Now, let's run CRI-O from the latest `main` commit. This can be done by either
building it from source, using the [static binary bundles](https://github.com/cri-o/packaging?tab=readme-ov-file#using-the-static-binary-bundles-directly)
or [the prerelease packages](https://github.com/cri-o/packaging?tab=readme-ov-file#usage).

To demonstrate this, I ran the `crio` binary from my command line using a single
node Kubernetes cluster via [`local-up-cluster.sh`](https://github.com/cri-o/cri-o?tab=readme-ov-file#running-kubernetes-with-cri-o).
Now that the cluster is up and running, let's try a pod without the annotation
running as seccomp `Unconfined`:
-->

现在，让我们从最新的`main`提交中运行 CRI-O。可以通过从源代码构建，使用[静态二进制包](https://github.com/cri-o/packaging?tab=readme-ov-file#using-the-static-binary-bundles-directly)
或[预发布软件包](https://github.com/cri-o/packaging?tab=readme-ov-file#usage)来实现这一点。

为了演示，我在我的命令行中运行了`crio`二进制文件，使用[`local-up-cluster.sh`](https://github.com/cri-o/cri-o?tab=readme-ov-file#running-kubernetes-with-cri-o)来启动单节点 Kubernetes 集群。现在集群已经启动并运行，让我们尝试一个没有注释的 Pod，以`Unconfined`模式运行 seccomp：

```shell
cat pod.yaml
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod
spec:
  containers:
    - name: container
      image: nginx:1.25.3
      securityContext:
        seccompProfile:
          type: Unconfined
```

```shell
kubectl apply -f pod.yaml
```

<!--
The workload is up and running:
-->

工作负载已启动并运行：

```shell
kubectl get pods
```

```console
NAME   READY   STATUS    RESTARTS   AGE
pod    1/1     Running   0          15s
```

<!--
And no seccomp profile got applied if I inspect the container using
[`crictl`](https://sigs.k8s.io/cri-tools):
-->

并且如果我使用[`crictl`](https://sigs.k8s.io/cri-tools)检查容器,则没有 seccomp 配置被应用：

```shell
export CONTAINER_ID=$(sudo crictl ps --name container -q)
sudo crictl inspect $CONTAINER_ID | jq .info.runtimeSpec.linux.seccomp
```

```console
null
```

<!--
Now, let's modify the pod to apply the profile `quay.io/crio/seccomp:v2` to the
container:
-->

现在，让我们修改 pod，将配置文件`quay.io/crio/seccomp:v2`应用到容器：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod
  annotations:
    seccomp-profile.kubernetes.cri-o.io/container: quay.io/crio/seccomp:v2
spec:
  containers:
    - name: container
      image: nginx:1.25.3
```

<!--
I have to delete and recreate the Pod, because only recreation will apply a new
seccomp profile:
-->

我必须删除并重新创建 Pod，因为只有重新创建才会应用新的 Seccomp 配置文件：

```shell
kubectl delete pod/pod
```

```console
pod "pod" deleted
```

```shell
kubectl apply -f pod.yaml
```

```console
pod/pod created
```

<!--
The CRI-O logs will now indicate that the runtime pulled the artifact:
-->

CRI-O 日志将表明运行时拉取了该工件：

```console
WARN[…] Allowed annotations are specified for workload [seccomp-profile.kubernetes.cri-o.io]
INFO[…] Found container specific seccomp profile annotation: seccomp-profile.kubernetes.cri-o.io/container=quay.io/crio/seccomp:v2  id=26ddcbe6-6efe-414a-88fd-b1ca91979e93 name=/runtime.v1.RuntimeService/CreateContainer
INFO[…] Pulling OCI artifact from ref: quay.io/crio/seccomp:v2  id=26ddcbe6-6efe-414a-88fd-b1ca91979e93 name=/runtime.v1.RuntimeService/CreateContainer
INFO[…] Retrieved OCI artifact seccomp profile of len: 18853  id=26ddcbe6-6efe-414a-88fd-b1ca91979e93 name=/runtime.v1.RuntimeService/CreateContainer
```

<!--
And the container is finally using the profile:
-->

并且容器最终使用了这个配置：

```shell
export CONTAINER_ID=$(sudo crictl ps --name container -q)
sudo crictl inspect $CONTAINER_ID | jq .info.runtimeSpec.linux.seccomp | head
```

```yaml
{
  "defaultAction": "SCMP_ACT_ERRNO",
  "defaultErrnoRet": 38,
  "architectures": [
    "SCMP_ARCH_X86_64",
    "SCMP_ARCH_X86",
    "SCMP_ARCH_X32"
  ],
  "syscalls": [
    {
```

<!--
The same would work for every container in the pod, if users replace the
`/container` suffix with the reserved name `/POD`, for example:
-->

如果用户将`/container`后缀替换为保留名称`/POD`，则每个容器中的相同操作都会生效，例如：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod
  annotations:
    seccomp-profile.kubernetes.cri-o.io/POD: quay.io/crio/seccomp:v2
spec:
  containers:
    - name: container
      image: nginx:1.25.3
```

<!--
### Using the annotation for a container image

While specifying seccomp profiles as OCI artifacts on certain workloads is a
cool feature, the majority of end users would like to link seccomp profiles to
published container images. This can be done by using a container image
annotation; instead of being applied to a Kubernetes Pod, the annotation is some
metadata applied at the container image itself. For example,
[Podman](https://podman.io) can be used to add the image annotation directly
during image build:
-->

### 对容器镜像使用注释

在某些工作负载中将 seccomp 配置文件指定为 OCI 工件是一个很酷的功能，但大多数终端用户更愿意将 seccomp 配置文件与已发布的容器镜像关联起来。这可以通过使用容器镜像注释来实现；而不是应用于 Kubernetes Pod，该注释是应用于容器镜像本身的一些元数据。例如，[Podman](https://podman.io) 可以在构建镜像时直接添加镜像注释：

```shell
podman build \
    --annotation seccomp-profile.kubernetes.cri-o.io=quay.io/crio/seccomp:v2 \
    -t quay.io/crio/nginx-seccomp:v2 .
```

<!--
The pushed image then contains the annotation:
-->

推送的镜像包含了这个注释：

```shell
skopeo inspect --raw docker://quay.io/crio/nginx-seccomp:v2 |
    jq '.annotations."seccomp-profile.kubernetes.cri-o.io"'
```

```console
"quay.io/crio/seccomp:v2"
```

<!--
If I now use that image in an CRI-O test pod definition:
-->

如果现在在一个 CRI-O 测试 Pod 定义中使用该镜像：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod
  # no Pod annotations set
spec:
  containers:
    - name: container
      image: quay.io/crio/nginx-seccomp:v2
```

<!--
Then the CRI-O logs will indicate that the image annotation got evaluated and
the profile got applied:
-->

然后，CRI-O 日志将显示镜像注释已被评估，并且配置文件已被应用：

```shell
kubectl delete pod/pod
```

```console
pod "pod" deleted
```

```shell
kubectl apply -f pod.yaml
```

```console
pod/pod created
```

```console
INFO[…] Found image specific seccomp profile annotation: seccomp-profile.kubernetes.cri-o.io=quay.io/crio/seccomp:v2  id=c1f22c59-e30e-4046-931d-a0c0fdc2c8b7 name=/runtime.v1.RuntimeService/CreateContainer
INFO[…] Pulling OCI artifact from ref: quay.io/crio/seccomp:v2  id=c1f22c59-e30e-4046-931d-a0c0fdc2c8b7 name=/runtime.v1.RuntimeService/CreateContainer
INFO[…] Retrieved OCI artifact seccomp profile of len: 18853  id=c1f22c59-e30e-4046-931d-a0c0fdc2c8b7 name=/runtime.v1.RuntimeService/CreateContainer
INFO[…] Created container 116a316cd9a11fe861dd04c43b94f45046d1ff37e2ed05a4e4194fcaab29ee63: default/pod/container  id=c1f22c59-e30e-4046-931d-a0c0fdc2c8b7 name=/runtime.v1.RuntimeService/CreateContainer
```

```shell
export CONTAINER_ID=$(sudo crictl ps --name container -q)
sudo crictl inspect $CONTAINER_ID | jq .info.runtimeSpec.linux.seccomp | head
```

```yaml
{
  "defaultAction": "SCMP_ACT_ERRNO",
  "defaultErrnoRet": 38,
  "architectures": [
    "SCMP_ARCH_X86_64",
    "SCMP_ARCH_X86",
    "SCMP_ARCH_X32"
  ],
  "syscalls": [
    {
```

<!--
For container images, the annotation `seccomp-profile.kubernetes.cri-o.io` will
be treated in the same way as `seccomp-profile.kubernetes.cri-o.io/POD` and
applies to the whole pod. In addition to that, the whole feature also works when
using the container specific annotation on an image, for example if a container
is named `container1`:
-->

对于容器镜像，注释 `seccomp-profile.kubernetes.cri-o.io` 将被视为与 `seccomp-profile.kubernetes.cri-o.io/POD` 相同，并适用于整个 Pod。除此之外，当在镜像上使用容器特定的注释时，整个功能也会起作用，例如，如果一个容器的名称是 `container1`：

```shell
skopeo inspect --raw docker://quay.io/crio/nginx-seccomp:v2-container |
    jq '.annotations."seccomp-profile.kubernetes.cri-o.io/container1"'
```

```console
"quay.io/crio/seccomp:v2"
```

<!--
The cool thing about this whole feature is that users can now create seccomp
profiles for specific container images and store them side by side in the same
registry. Linking the images to the profiles provides a great flexibility to
maintain them over the whole application's life cycle.
-->

这个整个功能的很酷之处在于，用户现在可以为特定的容器镜像创建 seccomp 配置文件，并将它们存储在同一个仓库中。将镜像与配置文件关联起来为在整个应用程序生命周期内维护它们提供了极大的灵活性。

<!--
### Pushing profiles using ORAS

The actual creation of the OCI object that contains a seccomp profile requires a
bit more work when using ORAS. I have the hope that tools like Podman will
simplify the overall process in the future. Right now, the container registry
needs to be [OCI compatible](https://oras.land/docs/compatible_oci_registries/#registries-supporting-oci-artifacts),
which is also the case for [Quay.io](https://quay.io). CRI-O expects the seccomp
profile object to have a container image media type
(`application/vnd.cncf.seccomp-profile.config.v1+json`), while ORAS uses
`application/vnd.oci.empty.v1+json` per default. To achieve all of that, the
following commands can be executed:
-->

### 使用 ORAS 推送配置文件

当使用 ORAS 时，实际创建包含 seccomp 配置文件的 OCI 对象需要更多工作。我希望像 Podman 这样的工具将来会简化整个过程。当前，容器注册表需要是[OCI 兼容的](https://oras.land/docs/compatible_oci_registries/#registries-supporting-oci-artifacts)，这也适用于[Quay.io](https://quay.io)。CRI-O 期望 seccomp 配置文件对象具有容器镜像媒体类型(`application/vnd.cncf.seccomp-profile.config.v1+json`)，而 ORAS 默认使用`application/vnd.oci.empty.v1+json`。为了实现所有这些，可以执行以下命令：

```shell
echo "{}" > config.json
oras push \
    --config config.json:application/vnd.cncf.seccomp-profile.config.v1+json \
     quay.io/crio/seccomp:v2 seccomp.json
```

<!--
The resulting image contains the `mediaType` that CRI-O expects. ORAS pushes a
single layer `seccomp.json` to the registry. The name of the profile does not
matter much. CRI-O will pick the first layer and check if that can act as a
seccomp profile.
-->

生成的镜像包含了 CRI-O 所期望的`mediaType`。ORAS 将单个层`seccomp.json`推送到注册表中。配置文件的名称并不太重要。CRI-O 将选择第一个层，并检查其是否可以作为 seccomp 配置文件。

<!--
## Future work

CRI-O internally manages the OCI artifacts like regular files. This provides the
benefit of moving them around, removing them if not used any more or having any
other data available than seccomp profiles. This enables future enhancements in
CRI-O on top of OCI artifacts, but also allows thinking about stacking seccomp
profiles as part of having multiple layers in an OCI artifact. The limitation
that it only works for `Unconfined` workloads for v1.30.x releases is something
different CRI-O would like to address in the future. Simplifying the overall
user experience by not compromising security seems to be the key for a
successful future of seccomp in container workloads.

The CRI-O maintainers will be happy to listen to any feedback or suggestions on
the new feature! Thank you for reading this blog post, feel free to reach out
to the maintainers via the Kubernetes [Slack channel #crio](https://kubernetes.slack.com/messages/CAZH62UR1)
or create an issue in the [GitHub repository](https://github.com/cri-o/cri-o).
-->

## 将来的工作

CRI-O 内部管理的 OCI 资源就像普通的文件一样。这提供了将它们移动、删除或在 seccomp 配置文件之外提供其他数据的好处。这使得在 OCI 工件之上对 CRI-O 进行未来增强成为可能，同时也考虑将 seccomp 配置文件堆叠作为 OCI 工件中的多个层的一部分。对于 v1.30.x 版本,仅适用于 Unconfined 工作负载的这个限制,是 CRI-O 希望未来解决的问题。通过不牺牲安全性来简化整体用户体验似乎是容器工作负载中 seccomp 的成功未来的关键。

CRI-O 的维护者将乐于听取有关这个新功能的任何反馈或建议！感谢您阅读这篇博客文章，欢迎通过 Kubernetes 的 [Slack 频道#crio](https://kubernetes.slack.com/messages/CAZH62UR1) 或在 [GitHub 存储库](https://github.com/cri-o/cri-o)中创建问题与维护者联系。
