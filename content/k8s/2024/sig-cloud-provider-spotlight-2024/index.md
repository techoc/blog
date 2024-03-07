---
title: '聚焦 SIG 云提供商'
description: 'Spotlight on SIG Cloud Provider'
date: 2024-03-02T16:30:31+08:00
image: "https://www4.bing.com//th?id=OHR.LeapingSquirrel_ZH-CN9112090462_1920x1080.jpg"
tags: ["Kubernetes","Kubernetes blog"]
categories: ["Kubernetes"]
slug: sig-cloud-provider-spotlight-2024
---

作者：Arujjwal Negi 

译者：[Rui Yang](https://github.com/techoc)

开发者使用 Kubernetes 服务最流行的方式之一是通过云提供商，但您有没有想过云提供商如何做到这一点？ Kubernetes 与各种云提供商集成的整个过程是如何发生的？ 为了回答这个问题，让我们关注一下 [SIG 云提供商](https://github.com/kubernetes/community/blob/master/sig-cloud-provider/README.md)。

SIG 云提供商致力于在 Kubernetes 和各种云提供商之间创建无缝集成。他们维持 Kubernetes 生态系统的公平性和开放性。通过制定清晰的标准和要求，以确保每个云提供商都能与 Kubernetes 良好协作。他们也负责配置集群组件以实现云提供商集成。

在聚焦 SIG 系列的这篇博客中，Arujjwal Negi 采访了 Michael McCune (Red Hat)，又名 elmiko，SIG 云提供商的联合主席，让我们来  深入了解该小组的工作。

## 导言
Arujjwal：让我们从了解你开始。你能给我们介绍一下你自己以及你是如何从事Kubernetes的吗？

Michael：嗨，我是Michael McCune，社区里的大多数人都叫我的名字elmiko。我做软件开发人员已经有很长时间了(我刚开始的时候Windows 3.1很流行!)，我第一次接触Kubernetes是作为机器学习和数据科学应用程序的开发人员;当时我所在的团队正在创建教程和示例，以演示在Kubernetes上使用Apache Spark等技术。也就是说，我对分布式系统感兴趣已经很多年了，当有机会加入一个直接在Kubernetes上工作的团队时，我立刻就抓住了它!

## 功能和工作
Arujjwal：您能否向我们介绍一下 SIG Cloud Provider 的业务及其运作方式？

Michael：SIG Cloud Provider 的成立是确保 Kubernetes 为所有基础设施提供商提供一个中立的接入点。 迄今为止，我们最大的任务是将 in-tree 云控制器提取并迁移到 out-of-tree 组件。 SIG 会定期召开会议，讨论进展情况和即将到来的任务，并解决出现的问题和错误。 此外，我们还充当云提供商子项目（例如cloud provider 框架、特定的云控制器实现和 [Konnectivity proxy](https://kubernetes.io/docs/tasks/extend-kubernetes/setup-konnectivity/)）的协调点。

Arujjwal：在阅读了项目的 [README](https://github.com/kubernetes/community/blob/master/sig-cloud-provider/README.md)之后，我了解到 SIG Cloud Provider 致力于 Kubernetes 与 cloud provider
的集成。那么整个过程是怎样进行的呢？

Michael：运行 Kubernetes 最常见的方法之一是将其部署到云环境（AWS、Azure、GCP 等）。 通常，云基础设施具有增强 Kubernetes 性能的功能，例如，通过为服务对象提供弹性负载平衡。 为了确保 Kubernetes 能够一致地使用特定于云的服务，Kubernetes 社区创建了云控制器来解决这些集成点。 云提供商可以使用 SIG 维护的框架或遵循 Kubernetes 代码和文档中定义的 API 指南来创建自己的控制器。 我想指出的一件事是，SIG Cloud Provider 不处理 Kubernetes 集群中节点的生命周期； 对于这些类型的主题，SIG Cluster Lifecycle 和 Cluster API 项目是更合适的场所。

## 重要的子项目

Arujjwal：这个 SIG 中有很多子项目。你能重点介绍一些最重要的内容以及它们的作用吗？

Michael：我认为当今两个最重要的子项目是 [cloud provider 框架](https://github.com/kubernetes/community/blob/master/sig-cloud-provider/README.md#kubernetes-cloud-provider) 和 [extraction/migration project](https://github.com/kubernetes/community/blob/master/sig-cloud-provider/README.md#cloud-provider-extraction-migration)。
cloud provider 框架是一个通用库，可帮助基础设施集成商为其基础设施构建云控制器。 该项目通常是新人加入 SIG 的起点。 extraction/migration project 是另一个大子项目，也是该框架存在的很大一部分原因。 了解一点历史可能有助于进一步解释：很长一段时间以来，Kubernetes 需要与底层基础设施进行某种集成，不一定是为了添加功能，而是为了了解实例终止等云事件。 云提供商集成内置于 Kubernetes 代码树中，因此创建了术语“树内”，详见该[文章](https://kaslin.rocks/out-of-tree/)。 社区认为在主 Kubernetes 源码树中维护特定于提供者的代码的活动是不可取的。 社区的决定激发了 extraction and migration project 的创建，以删除“树内”云控制器，转而使用“树外”组件。

