---
title: "浅谈容器启动命令"
description: "Dockerfile 中的 ENTRYPOINT 和 CMD 与 Kubernetes 中的 command 和 args"
date: 2024-03-13T17:05:28+08:00
image: "https://cn.bing.com/th?id=OHR.AyutthayaTree_ZH-CN8075870220_1920x1080.jpg&rf=LaDigue_1920x1080.jpg&pid=hp"
comments: true
tags: ["Kubernetes"]
categories: ["Kubernetes"]
slug: "talk-about-define-command-argument-container"
---

## 什么是容器启动命令

在创建 POD 时，我们可以为容器指定需要执行的启动命令以及参数。如果要设置命令，就填写在配置文件的 command 字段下，如果要设置命令的参数，就填写在配置文件的 args 字段下。 一旦 Pod 创建完成，该命令及其参数就无法再进行更改了。

对于 Kubernetes 而言，配置文件中的 `command` 和 `args` 分别对应的是 Dockerfile 中的 `ENTRYPOINT` 和 `CMD` 命令。

这意味着当你在 Kubernetes Pod 中指定 `command` 时，它会覆盖 Docker 镜像中定义的 `ENTRYPOINT` 。同样，当你指定 `args` 时，它会覆盖 Docker 镜像中定义的 `CMD` 。但会有一些特殊情况，比如只指定配置 `command` 的时候，只指定配置 `args` 的时候，下面会对这些情况进行详细说明

在了解这几个特殊情况之前，我们先来了解下 Dockerfile 中的 `ENTRYPOINT` 和 `CMD`。

对于 Dockerfile 中的 `ENTRYPOINT` 和 `CMD` 容器应用进程 = `ENTRYPOINT` + `CMD`。

`ENTRYPOINT` 定义了容器启动时运行的命令。它可以是一个可执行文件或者一个脚本，例如 [nginx 容器的 Dockerfile](https://hub.docker.com/layers/library/nginx/latest/images/sha256-52478f8cd6a142fd462f0a7614a7bb064e969a4c083648235d6943c786df8cc7?context=explore)下的 `ENTRYPOINT` 为 `/docker-entrypoint.sh`。如果在 Dockerfile 中没有显式配置 `ENTRYPOINT`，Docker 会使用一个默认的 `ENTRYPOINT`。这个默认的 `ENTRYPOINT` 是 `/bin/sh -c`。

需要注意的是当一个 Dockerfile 中同时存在多条 `ENTRYPOINT` 时，只有最后一条`ENTRYPOINT`会生效。

`CMD` 定义了容器启动时运行的命令的参数。`CMD` 提供了 `ENTRYPOINT` 的默认参数。如果没有提供 `CMD`，那么 `ENTRYPOINT` 需要是一个完整的命令。如果提供了 `CMD`，它可以作为 `ENTRYPOINT` 的参数。反之如果没有 `ENTRYPOINT`，则 `CMD` 需要是一个完整命令。

## 使用的几种情况

1. 替换而非追加：在 Kubernetes 中，当你设置 `command` 或 `args` 时，你是在替换 `Docker` 镜像中的 `ENTRYPOINT` 和 `CMD`，而不是在它们后面追加命令或参数。
2. 只设置 `command`：当设置 `command` 时，`Docker` 镜像中的 `ENTRYPOINT` 将被替换，而 `CMD` 将被忽略。
3. 只设置 `args`：当设置 `args` 时，`Docker` 镜像中的 `CMD` 将被替换，而 `ENTRYPOINT` 将被忽略。

<table align="center">
    <tr>
        <th colspan="2" style="text-align:center">k8s pod</th> 
        <th colspan="2" style="text-align:center">docker</th> 
        <th rowspan="2" style="text-align:center">最后生效的命令</th> 
    </tr>
    <tr>
        <td style="text-align:center">command</td>
        <td style="text-align:center">args</td>
        <td style="text-align:center">entrypoint</td>
        <td style="text-align:center">cmd</td>
    </tr>
    <tr>
        <td style="text-align:center">配置</td>  
        <td style="text-align:center">配置</td>  
        <td style="text-align:center">配置</td>  
        <td style="text-align:center">配置</td>  
        <td style="text-align:center">entrypoint+args</td>  
    </tr>
    <tr>
        <td style="text-align:center">配置</td>  
        <td style="text-align:center">不配置</td>  
        <td style="text-align:center">配置</td>  
        <td style="text-align:center">配置</td>  
        <td style="text-align:center">command</td>  
    </tr>
    <tr>
        <td style="text-align:center">不配置</td>  
        <td style="text-align:center">配置</td>  
        <td style="text-align:center">配置</td>  
        <td style="text-align:center">配置</td>  
        <td style="text-align:center"style="text-align:center">entrypoint+args</td>  
    </tr>
    <tr>
        <td style="text-align:center">不配置</td>  
        <td style="text-align:center">不配置</td>  
        <td style="text-align:center">配置</td>  
        <td style="text-align:center">配置</td>  
        <td style="text-align:center">entrypoint+cmd</td>  
    </tr>
    <tr>
        <td style="text-align:center">不配置</td>  
        <td style="text-align:center">配置</td>  
        <td style="text-align:center">不配置</td>  
        <td style="text-align:center">配置</td>  
        <td style="text-align:center">args</td>  
    </tr>
    <tr>
        <td style="text-align:center">不配置</td>  
        <td style="text-align:center">配置</td>  
        <td style="text-align:center">配置</td>  
        <td style="text-align:center">不配置</td>  
        <td style="text-align:center">entrypoint+args</td>  
    </tr>
</table>

> 总而言之 Kubernetes 中的`command`对应于 Docker 的`ENTRYPOINT`，而`args`对应于 Docker 的`CMD`，并且在 Kubernetes Pod 定义中指定这些会覆盖 Docker 镜像中的相应设置。

## 示例

有一个 例如 [Bash 镜像](https://hub.docker.com/layers/library/bash/devel-alpine3.19/images/sha256-6083a63aa4e4efb05ce27e05a13c389f99a1d39938f923f9cc53907132d5b150?context=explore) ，它有一个 `ENTRYPOINT` 和 `CMD`，如下所示：

```dockerfile
ENTRYPOINT ["docker-entrypoint.sh"]
CMD ["bash"]
```

在这个 Dockerfile ，默认的`ENTRYPOINT`是`docker-entrypoint.sh`，而`bash`是传递给它的默认`CMD`。

### 只设置 `command`，不设置 `args`

Kubernetes Pod 的配置如下：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: bash
spec:
  containers:
    - name: python-container
      image: bash
      command: ["bash", "-c", "echo hello"]
      # args: ["python3 --version"]
  restartPolicy: OnFailure
```

可以看到输出：

```shell
# k logs bash
hello
```

即所执行的命令为 `bash -c echo hello bash`(Kubernetes yaml 中的 `command`+ Dockerfile 中的 `CMD`)。

### 只设置 `args`，不设置 `command`

Kubernetes Pod 的配置如下：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: bash
spec:
  containers:
    - name: python-container
      image: bash
      # command: ["bash", "-c", "echo hello"]
      args: ["python3 --version"]
  restartPolicy: OnFailure
```

可以看到输出：

```shell
# k logs bash
/usr/local/bin/docker-entrypoint.sh: line 11: exec: python3 --version: not found
```

即所执行的命令为 `docker-entrypoint.sh python3 --version`(Dockerfile 中的 `ENTRYPOINT` + Kubernetes yaml 中的 `args`)。

### 设置 `command` 和 `args`

Kubernetes Pod 的配置如下：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: bash
spec:
  containers:
    - name: python-container
      image: bash
      command: ["bash", "-c"]
      args: ["echo hello"]
  restartPolicy: OnFailure
```

可以看到输出：

```shell
# k logs bash
hello
```

即所执行的命令为 `bash -c echo hello`(Kubernetes yaml 中的 `command` + Kubernetes yaml 中的 `args`)。

### 不设置 `command` 和 `args`

即直接运行 Docker 镜像中的 `ENTRYPOINT` 和 `CMD`。
