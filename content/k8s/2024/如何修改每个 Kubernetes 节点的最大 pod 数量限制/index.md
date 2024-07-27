---
title: "如何修改每个 Kubernetes 节点的最大 pod 数量限制"
description: "如何修改每个 Kubernetes 节点的最大 pod 数量限制"
date: 2024-03-13T17:05:28+08:00
image: 2024-07-15-22-51-03.png
comments: true
tags: ["Kubernetes"]
categories: ["Kubernetes"]
slug: "how-to-increase-the-number-of-pods-limit-per-kubernetes-node"
---

[Kubernetes](https://kubernetes.io/) 是一个开源的容器编排系统，用于自动化容器化应用的部署、扩展和运行。Kubernetes
集群中的每个节点都有一个 pod 数量限制，默认为 110。

你可以通过修改 kubelet 的配置文件中的 `maxPods` 参数来修改 pod
数量限制。具体的[配置文件](https://kubernetes.io/zh-cn/docs/reference/config-api/kubelet-config.v1beta1/)可以查看。

1. 找到 kubelet 的配置文件路径。可以执行 `systemctl status kubelet` 命令查看：

```shell
[root@k8s-node01 kubelet]# systemctl status kubelet
● kubelet.service - kubelet: The Kubernetes Node Agent
   Loaded: loaded (/usr/lib/systemd/system/kubelet.service; enabled; vendor preset: disabled)
  Drop-In: /usr/lib/systemd/system/kubelet.service.d
           └─10-kubeadm.conf
   Active: active (running) since 一 2024-07-15 22:35:01 CST; 53min ago
     Docs: https://kubernetes.io/docs/
 Main PID: 114544 (kubelet)
    Tasks: 11
   Memory: 37.4M
   CGroup: /system.slice/kubelet.service
           └─114544 /usr/bin/kubelet --bootstrap-kubeconfig=/etc/kubernetes/bootstrap-kubelet.conf --kubeconfig=/etc/kubernetes/kubelet.conf --config=/var/lib/kubelet/c...
```

可以看到，kubelet 的配置文件路径为 `/var/lib/kubelet/config.yaml`

2. 编辑 kubelet 的配置文件，将 `maxPods` 参数的值修改为你需要的 pod 数量限制。例如，将 `maxPods` 的值修改为 200：

```yaml
apiVersion: kubelet.config.k8s.io/v1beta1
authentication:
  anonymous:
    enabled: false
  webhook:
    cacheTTL: 0s
    enabled: true
  x509:
    clientCAFile: /etc/kubernetes/pki/ca.crt
authorization:
  mode: Webhook
  webhook:
    cacheAuthorizedTTL: 0s
    cacheUnauthorizedTTL: 0s
cgroupDriver: systemd
clusterDNS:
  - 20.1.0.10
clusterDomain: cluster.local
containerRuntimeEndpoint: ""
cpuManagerReconcilePeriod: 0s
evictionPressureTransitionPeriod: 0s
fileCheckFrequency: 0s
healthzBindAddress: 127.0.0.1
healthzPort: 10248
httpCheckFrequency: 0s
imageMinimumGCAge: 0s
kind: KubeletConfiguration
logging:
  flushFrequency: 0
  options:
    json:
      infoBufferSize: "0"
  verbosity: 0
memorySwap: {}
nodeStatusReportFrequency: 0s
nodeStatusUpdateFrequency: 0s
rotateCertificates: true
runtimeRequestTimeout: 0s
shutdownGracePeriod: 0s
shutdownGracePeriodCriticalPods: 0s
staticPodPath: /etc/kubernetes/manifests
streamingConnectionIdleTimeout: 0s
syncFrequency: 0s
volumeStatsAggPeriod: 0s
maxPods: 200
```

3. 重启 kubelet 服务，使修改生效 `systemctl restart kubelet`
4. 验证修改: `kubectl describe node k8s-node01 | grep maxPods`
