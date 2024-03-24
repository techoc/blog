---
title: "NodePort和HostPort与HostNetwork之间的区别"
description: "Kubernetes 集群外部访问的几种访问方式之间的区别"
date: 2024-03-20T10:20:15+08:00
image: "https://s.cn.bing.net/th?id=OHR.WhiteEyes_ZH-CN1130380430_1920x1080.jpg&rf=LaDigue_1920x1080.jpg&pid=hp"
tags: ["Kubernetes"]
categories: ["Kubernetes"]
slug: "nodeport-hostport-hostnetwork"
---

## NodePort

`NodePort` 在 `Kubernetes` 里是一个广泛应用的服务暴露方式。 `Kubernetes` 中的 `service` 默认情况下都是使用的 `ClusterIP` 这种类型，这样的 `service` 会产生一个 `ClusterIP` ，这个 `IP` 只能在集群内部访问，要想让外部能够直接访问 `service` ，需要将 `service type` 修改为 `nodePort` 。

如下就是一个 NodePort 的示例：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      nodeSelector:
        nginx: ok
      containers:
        - name: nginx
          image: nginx:stable
          command: ["nginx", "-g", "daemon off;"]
---
kind: Service
apiVersion: v1
metadata:
  name: nginx-svc
spec:
  type: NodePort
  ports:
    - port: 80
      nodePort: 30001
  selector:
    name: nginx
```

```shell
$ kube get svc -o wide
NAME         TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)        AGE    SELECTOR
kubernetes   ClusterIP   10.96.0.1      <none>        443/TCP        176d   <none>
nginx-svc    NodePort    10.110.21.50   <none>        80:30001/TCP   26s    app=nginx

$ kube get pod -o wide
NAME                                    READY   STATUS    RESTARTS   AGE   IP            NODE           NOMINATED NODE   READINESS GATES
clubproxy-deployment-55c958bf67-cg424   1/1     Running   0          11h   10.244.0.56   k8s-master01   <none>           <none>
nginx-deployment-c9fd6b54f-bdr8l        1/1     Running   0          18s   10.244.0.58   k8s-master01   <none>           <none>

$ sudo netstat -lpn | grep 30001
tcp        0      0 0.0.0.0:30001           0.0.0.0:*               LISTEN      15954/kube-proxy
```

如上配置，可以通过 `{pod_ip}/index.html`、`{cluster_ip}/index.html`、`{node_ip}:30001/index.html` 访问 nginx。node 节点上可以看到 `kube-proxy` 进程在监听 30001 端口，用于接收从主机网络进入的外部流量。最后，通过 `iptables` 规则将 `{node_ip}:30001` 映射到 `{pod_ip}:容器端口`。

## HostPort

`hostPort` 是将容器端口与宿主节点上的端口建立映射关系，这样用户就可以通过宿主机的 `IP` 加上来访问 `Pod` 了。

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
        - name: nginx
          image: nginx:stable
          command: ["nginx", "-g", "daemon off;"]
          ports:
            - containerPort: 80
              hostPort: 81
```

```shell
$ kube get pod -o wide|grep nginx
nginx-deployment-ff549cb9d-5t4bp        1/1     Running   0          5m8s    10.244.0.40   k8s-master01   <none>           <none>

$ sudo iptables -L -nv -t nat| grep "0.244.0.40:80"
    0     0 DNAT       tcp  --  *      *       0.0.0.0/0            0.0.0.0/0            tcp dpt:81 to:10.244.0.40:80
```

如上配置，既可以通过 `curl 10.244.0.40/index.html` 访问 nginx ，也可以通过 `curl {node_ip}:81/index.html`访问`nginx`。可以看到 `iptables` 会添加一条 `DNAT` 规则，将宿主机的 81 端口映射到 Pod 的 80 端口。

## HostNetwork

这种网络模式 ，相当于 `docker run --net=host`，采用宿主机的网络命名空间。此时，Pod 的 ip 为 node 节点的 ip，Pod 的端口需要保持不与宿主机上的 port 端口发生冲突。

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      hostNetwork: true
      containers:
        - name: nginx
          image: nginx:stable
          command: ["nginx", "-g", "daemon off;"]
```

```shell
$ sudo ps aux| grep nginx
root     20126  0.0  0.0  10640  3512 ?        Ss   10:17   0:00 nginx: master process nginx -g daemon off;
101      20138  0.0  0.0  11044  1524 ?        S    10:17   0:00 nginx: worker process
101      20139  0.0  0.0  11044  1524 ?        S    10:17   0:00 nginx: worker process
101      20140  0.0  0.0  11044  1524 ?        S    10:17   0:00 nginx: worker process
101      20141  0.0  0.0  11044  1524 ?        S    10:17   0:00 nginx: worker process

$ sudo netstat -lpn| grep ":80 "
tcp        0      0 0.0.0.0:80              0.0.0.0:*               LISTEN      20126/nginx: master
```

如上配置，可以直接通过 `curl {node_ip}/index.html` 访问 nginx ，并且由于 Pod 采用的是宿主机的网络命名空间，因此宿主机可以直接看到 nginx master 进程正在监听 80 端口。

## HostPort 和 HostNetwork 的异同

相同点：

1. hostPort 与 hostNetwork 本质上都是终端用户能访问到 pod 中的业务
2. 访问的时候，只能用 pod 所在宿主机 IP + 容器端口或 hostport 端口 进行访问

不同点：

1. 网络地址空间不同。hostport 使用 CNI 分配的地址，hostNetwork 使用宿主机网络地址空间
2. hostport 宿主机不生成端口，hostNetwork 宿主机生成端口
3. hostport 通过 iptable 防火墙的 nat 表进行转发，hostNetwork 直接通过主机端口到容器中
4. 定义的路径不同。`deploy.spec.template.spec.containers.ports.hostPort` 与 `deploy.spec.template.spec.hostNetwork`
5. 优先级不同，hostNetwork 高于 hostPort
