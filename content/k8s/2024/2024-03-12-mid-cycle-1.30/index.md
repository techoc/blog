---
title: "Kubernetes v1.30 初探"
description: "A Peek at Kubernetes v1.30"
date: 2024-03-16T16:00:05+08:00
image:
slug: kubernetes-1-30-upcoming-changes
---

<!--

---
layout: blog
title: 'A Peek at Kubernetes v1.30'
date: 2024-03-12
slug: kubernetes-1-30-upcoming-changes
---

-->

<!--
**Authors:** Amit Dsouza, Frederick Kautz, Kristin Martin, Abigail McCarthy, Natali Vlatko
-->

**作者:** Amit Dsouza, Frederick Kautz, Kristin Martin, Abigail McCarthy, Natali Vlatko

**译者:** [Rui Yang](https://github.com/techoc)

<!--
## A quick look: exciting changes in Kubernetes v1.30

It's a new year and a new Kubernetes release. We're halfway through the release cycle and
have quite a few interesting and exciting enhancements coming in v1.30. From brand new features
in alpha, to established features graduating to stable, to long-awaited improvements, this release
has something for everyone to pay attention to!

To tide you over until the official release, here's a sneak peek of the enhancements we're most
excited about in this cycle!
-->

## 快速预览：Kubernetes v1.30 的精彩新功能

新的一年，新的 Kubernetes 版本。我们已经进行了一半的发布周期，并且在 v1.30 中有许多有趣和令人兴奋的增强功能。从全新的 alpha 版本功能，到已有的功能升级到稳定版，再到期待已久的改进，这个版本对每个人都有值得关注的东西！

在正式发布之前，让我们先来看一看这个周期中我们最为期待的增强功能的预览！

<!--
## Major changes for Kubernetes v1.30

### Structured parameters for dynamic resource allocation ([KEP-4381](https://kep.k8s.io/4381))

[Dynamic resource allocation](/docs/concepts/scheduling-eviction/dynamic-resource-allocation/) was
added to Kubernetes as an alpha feature in v1.26. It defines an alternative to the traditional
device-plugin API for requesting access to third-party resources. By design, dynamic resource
allocation uses parameters for resources that are completely opaque to core Kubernetes. This
approach poses a problem for the Cluster Autoscaler (CA) or any higher-level controller that
needs to make decisions for a group of pods (e.g. a job scheduler). It cannot simulate the effect of
allocating or deallocating claims over time. Only the third-party DRA drivers have the information
available to do this.
-->

## Kubernetes v1.30 的主要变更

### 为动态资源分配构建结构化参数 ([KEP-4381](https://kep.k8s.io/4381))

在 Kubernetes v1.26 中作为 alpha 功能添加了[动态资源分配](/docs/concepts/scheduling-eviction/dynamic-resource-allocation/)。它定义了一种替代传统设备插件 API 的方法，用于请求访问第三方资源。根据设计，动态资源分配使用对于核心 Kubernetes 完全不透明的资源参数。这种方法对于集群自动缩放器（CA）或任何需要为一组 Pod（例如作业调度器）做出决策的更高级别控制器来说是个问题。它无法模拟随时间分配或释放资源请求的效果。只有第三方 DRA 驱动程序才有可用的信息来执行此操作。

<!--
​​Structured Parameters for dynamic resource allocation is an extension to the original
implementation that addresses this problem by building a framework to support making these claim
parameters less opaque. Instead of handling the semantics of all claim parameters themselves,
drivers could manage resources and describe them using a specific "structured model" pre-defined by
Kubernetes. This would allow components aware of this "structured model" to make decisions about
these resources without outsourcing them to some third-party controller. For example, the scheduler
could allocate claims rapidly without back-and-forth communication with dynamic resource
allocation drivers. Work done for this release centers on defining the framework necessary to enable
different "structured models" and to implement the "named resources" model. This model allows
listing individual resource instances and, compared to the traditional device plugin API, adds the
ability to select those instances individually via attributes.
-->

“动态资源分配的结构化参数”是对原始实现的扩展，通过构建一个框架来支持使这些声明参数不再那么不透明，从而解决了这个问题。驱动程序不再直接处理所有资源参数的语义，而是可以使用由 Kubernetes 预先定义的特定的“结构化模型”来管理资源并描述它们。这将允许了解这个“结构化模型”的组件在不将资源外包给某些第三方控制器的情况下做出关于这些资源的决策。例如，调度器可以快速分配资源而无需与动态资源分配驱动程序进行来回通信。这个版本的工作集中在定义必要的框架来启用不同的“结构化模型”以及实现“命名资源”模型上。这个模型允许列出各个资源实例，并且与传统的设备插件 API 相比，增加了通过属性单独选择这些实例的能力。

<!--
### Node memory swap support ([KEP-2400](https://kep.k8s.io/2400))

In Kubernetes v1.30, memory swap support on Linux nodes gets a big change to how it works - with a
strong emphasis on improving system stability. In previous Kubernetes versions, the `NodeSwap`
feature gate was disabled by default, and when enabled, it used `UnlimitedSwap` behavior as the
default behavior. To achieve better stability, `UnlimitedSwap` behavior (which might compromise node
stability) will be removed in v1.30.
-->

### 节点内存交换支持 ([KEP-2400](https://kep.k8s.io/2400))

在 Kubernetes v1.30 中，Linux 节点上的内存交换支持得到了重大变化，重点是改善系统稳定性。在先前的 Kubernetes 版本中，`NodeSwap` 特性门默认处于禁用状态，启用时，默认行为是使用 `UnlimitedSwap`。为了实现更好的稳定性，在 v1.30 中将删除 `UnlimitedSwap` 行为（可能会影响节点稳定性）。

<!--
The updated, still-beta support for swap on Linux nodes will be available by default. However, the
default behavior will be to run the node set to `NoSwap` (not `UnlimitedSwap`) mode. In `NoSwap`
mode, the kubelet supports running on a node where swap space is active, but Pods don't use any of
the page file. You'll still need to set `--fail-swap-on=false` for the kubelet to run on that node.
However, the big change is the other mode: `LimitedSwap`. In this mode, the kubelet actually uses
the page file on that node and allows Pods to have some of their virtual memory paged out.
Containers (and their parent pods) do not have access to swap beyond their memory limit, but the
system can still use the swap space if available.
-->

更新后的仍处于测试阶段的 Linux 节点上的交换支持将默认可用。然而，默认行为将是将节点设置为 `NoSwap` 模式（而不是 `UnlimitedSwap`）。在 `NoSwap` 模式下，kubelet 支持在交换空间处于活动状态的节点上运行，但 Pod 不使用任何页面文件。你仍然需要设置 `--fail-swap-on=false` 让 kubelet 在该节点上运行。然而，另一种模式的变化很大：`LimitedSwap`。在这种模式下，kubelet 实际上会使用该节点上的页面文件，并允许 Pod 将部分虚拟内存分页出去。容器（及其父 Pod）无法超出其内存限制访问交换，但系统仍然可以使用交换空间（如果可用的话）。

<!--
Kubernetes' Node special interest group (SIG Node) will also update the documentation to help you
understand how to use the revised implementation, based on feedback from end users, contributors,
and the wider Kubernetes community.

Read the previous [blog post](/blog/2023/08/24/swap-linux-beta/) or the [node swap
documentation](/docs/concepts/architecture/nodes/#swap-memory) for more details on
Linux node swap support in Kubernetes.
-->

Kubernetes 的节点特别兴趣小组（SIG Node）还将根据来自最终用户、贡献者和更广泛的 Kubernetes 社区的反馈，更新文档，以帮助你理解如何使用修订后的实现。

要获取有关 Kubernetes 中 Linux 节点交换支持的更多详细信息，请阅读以前的[博客文章](https://kubernetes.io/blog/2023/08/24/swap-linux-beta/)或[节点交换文档](https://kubernetes.io/docs/concepts/architecture/nodes/#swap-memory)。

<!--
### Support user namespaces in pods ([KEP-127](https://kep.k8s.io/127))

[User namespaces](/docs/concepts/workloads/pods/user-namespaces) is a Linux-only feature that better
isolates pods to prevent or mitigate several CVEs rated high/critical, including
[CVE-2024-21626](https://github.com/opencontainers/runc/security/advisories/GHSA-xr7r-f8xq-vfvv),
published in January 2024. In Kubernetes 1.30, support for user namespaces is migrating to beta and
now supports pods with and without volumes, custom UID/GID ranges, and more!
-->

### 支持在 Pods 中使用用户命名空间 ([KEP-127](https://kep.k8s.io/127))

[用户命名空间](https://kubernetes.io/docs/concepts/workloads/pods/user-namespaces)是一个仅适用于 Linux 的功能，它更好地隔离 Pods，以防止或减轻几个被评为高/严重的 CVE,包括[CVE-2024-21626](https://github.com/opencontainers/runc/security/advisories/GHSA-xr7r-f8xq-vfvv)，于 2024 年 1 月发布。在 Kubernetes 1.30 中，对用户命名空间的支持正在迁移到 beta 版，并且现在支持具有和不具有卷、自定义 UID/GID 范围等功能的 Pods！

<!--
### Structured authorization configuration ([KEP-3221](https://kep.k8s.io/3221))

Support for [structured authorization
configuration](/docs/reference/access-authn-authz/authorization/#configuring-the-api-server-using-an-authorization-config-file)
is moving to beta and will be enabled by default. This feature enables the creation of
authorization chains with multiple webhooks with well-defined parameters that validate requests in a
particular order and allows fine-grained control – such as explicit Deny on failures. The
configuration file approach even allows you to specify [CEL](/docs/reference/using-api/cel/) rules
to pre-filter requests before they are dispatched to webhooks, helping you to prevent unnecessary
invocations. The API server also automatically reloads the authorizer chain when the configuration
file is modified.
-->

### 结构化授权配置 ([KEP-3221](https://kep.k8s.io/3221))

对[结构化授权配置](https://kubernetes.io/docs/reference/access-authn-authz/authorization/#configuring-the-api-server-using-an-authorization-config-file)的支持正在迁移到 beta 版，并将默认启用。该功能使你能够创建具有多个 Webhook 的授权链，具有明确定义的参数，按特定顺序验证请求，并允许细粒度控制，例如在失败时明确拒绝。配置文件方法甚至允许你指定[CEL](https://kubernetes.io/docs/reference/using-api/cel/)规则，以在将请求分派给 Webhook 之前对其进行预过滤，帮助你防止不必要的调用。当配置文件被修改时，API 服务器还会自动重新加载授权链。

<!--
You must specify the path to that authorization configuration using the `--authorization-config`
command line argument. If you want to keep using command line flags instead of a
configuration file, those will continue to work as-is. To gain access to new authorization webhook
capabilities like multiple webhooks, failure policy, and pre-filter rules, switch to putting options
in an `--authorization-config` file. From Kubernetes 1.30, the configuration file format is
beta-level, and only requires specifying `--authorization-config` since the feature gate is enabled by
default. An example configuration with all possible values is provided in the [Authorization
docs](/docs/reference/access-authn-authz/authorization/#configuring-the-api-server-using-an-authorization-config-file).
For more details, read the [Authorization
docs](/docs/reference/access-authn-authz/authorization/#configuring-the-api-server-using-an-authorization-config-file).
-->

你必须使用 `--authorization-config` 命令行参数指定授权配置的路径。如果你想继续使用命令行标志而不是配置文件，那么它们将继续像原样工作。要访问新的授权 Webhook 功能，如多个 Webhook、失败策略和预过滤规则，请切换到将选项放入 `--authorization-config` 文件中。从 Kubernetes 1.30 开始，配置文件格式处于 beta 级别，只需要指定 `--authorization-config`，因为默认情况下启用了功能门。在[授权文档](https://kubernetes.io/docs/reference/access-authn-authz/authorization/#configuring-the-api-server-using-an-authorization-config-file)中提供了一个包含所有可能值的示例配置。更多详细信息，请阅读[授权文档](https://kubernetes.io/docs/reference/access-authn-authz/authorization/#configuring-the-api-server-using-an-authorization-config-file)。

<!--
### Container resource based pod autoscaling ([KEP-1610](https://kep.k8s.io/1610))

Horizontal pod autoscaling based on `ContainerResource` metrics will graduate to stable in v1.30.
This new behavior for HorizontalPodAutoscaler allows you to configure automatic scaling based on the
resource usage for individual containers, rather than the aggregate resource use over a Pod. See our
[previous article](/blog/2023/05/02/hpa-container-resource-metric/) for further details, or read
[container resource metrics](/docs/tasks/run-application/horizontal-pod-autoscale/#container-resource-metrics).
-->

### 基于容器资源的 Pod 自动缩放 ([KEP-1610](https://kep.k8s.io/1610))

基于 `ContainerResource` 指标的水平 Pod 自动缩放将在 v1.30 中稳定发布。这种新的 HorizontalPodAutoscaler 行为允许你根据单个容器的资源使用情况配置自动缩放，而不是基于 Pod 的总体资源使用情况。欲了解更多详情，请参阅我们的[先前文章](https://kubernetes.io/blog/2023/05/02/hpa-container-resource-metric/)，或阅读[容器资源指标](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/#container-resource-metrics)。

<!--
### CEL for admission control ([KEP-3488](https://kep.k8s.io/3488))

Integrating Common Expression Language (CEL) for admission control in Kubernetes introduces a more
dynamic and expressive way of evaluating admission requests. This feature allows complex,
fine-grained policies to be defined and enforced directly through the Kubernetes API, enhancing
security and governance capabilities without compromising performance or flexibility.
-->

### CEL 用于准入控制 ([KEP-3488](https://kep.k8s.io/3488))

在 Kubernetes 中集成通用表达式语言（CEL）用于准入控制引入了一种更动态和富有表现力的方式来评估准入请求。这个功能允许通过 Kubernetes API 直接定义和强制执行复杂、细粒度的策略，增强了安全性和治理能力，而不会影响性能或灵活性。

<!--
CEL's addition to Kubernetes admission control empowers cluster administrators to craft intricate
rules that can evaluate the content of API requests against the desired state and policies of the
cluster without resorting to Webhook-based access controllers. This level of control is crucial for
maintaining the integrity, security, and efficiency of cluster operations, making Kubernetes
environments more robust and adaptable to various use cases and requirements. For more information
on using CEL for admission control, see the [API
documentation](/docs/reference/access-authn-authz/validating-admission-policy/) for
ValidatingAdmissionPolicy.

We hope you're as excited for this release as we are. Keep an eye out for the official release
blog in a few weeks for more highlights!
-->

CEL 的添加到 Kubernetes 准入控制赋予了集群管理员制定复杂规则的能力，这些规则可以评估 API 请求的内容与集群的期望状态和策略是否匹配，而无需求助于基于 Webhook 的访问控制器。这种控制水平对于维护集群操作的完整性、安全性和效率至关重要，使 Kubernetes 环境更加健壮，并能够适应各种用例和要求。有关使用 CEL 进行准入控制的更多信息，请参阅[API 文档](https://kubernetes.io/docs/reference/access-authn-authz/validating-admission-policy/)中的 ValidatingAdmissionPolicy 部分。

我们希望你和我们一样对这个版本的发布感到兴奋。请密切关注几周后发布的官方发布博客，了解更多亮点！

> 原文地址: https://kubernetes.io/blog/2024/03/12/kubernetes-1-30-upcoming-changes/
> 官方译文: https://kubernetes.io/zh-cn/blog/2024/03/12/kubernetes-1-30-upcoming-changes/
