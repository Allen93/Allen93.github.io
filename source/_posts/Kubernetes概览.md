---
title: Kubernetes概览
date: 2017-12-23 14:33:03
tags: Kubernetes
---
## Kubernetes概览
### Kubernetes
Kubernetes（简称 K8S）是一个生产级，开源，用于在集群级别部署，扩缩容，管理容器应用的基础组件。启发于 [borg](https://research.google.com/pubs/pub44843.html)。它致力于消除编排物理/虚拟层计算的，网络，存储等基础设备的负担，帮助运维和开发聚焦于容器本身。 K8S也为自定义工作流和自动化，提供了一个稳定，跨平台可移植的基础平台。
K8S通过了 Pods 和 Label 的方式，提供一组容器紧耦合和松耦合管理方式，帮助一组容器组合为的应用。


### 设计目标
* 可移植 K8S
* 满足通用需求
* 边界清晰，不做过多事情
* 灵活性
* 拓展性
* 自动化
* 架构演进


### 结构
典型的 Master-Slave 结构， Controllers 负责调度管理，通过分布式协议保证数据一致。 Worker 作为工作节点，运行应用容器。

### Controllers
提供一组 REST API，用于操作资源，K8S 的 api 提供了类 IAAS 的容器概念的操作，比如 PODS，Services，Ingress，以及一组生命周期的 API 用于这些支持基础类型资源的编排，比如 RC，RS，Deployment。
用户和管控组件都操作同样的 API，大部分资源包含元数据（labels，注解）以及目标状态（默认值，当前状态）等。Controllers会持续工作，操作资源到目标状态，并且上报当前状态。

#### API Server
* APi Server 提供 K8S 的 api 服务， 它只是一个 相对简单的 web Server，大部分业务逻辑在其他通过插件式组件中实现，它的主要处理 REST 操作，校验，并且维护 ETCD （或者其他存储组件）中存储的对象。 由于一些原因，不提供类似事务的多个资源原子操作。
* API Server 提供基本的 API 机制， 
	* 包括 REST 语法， watch，durability and consistency guarantees， 版本，默认值，校验。
	* 内置权限控制，同步的权限控制hook，异步的资源初始化操作。
	* API 注册和发现。

* API Server扮演集群网关的角色，一般来说既能被外访问，也能被集群中节点，容器访问。 客户端通过 API Server 认证，把它作为访问 node / pod 的隧道。

#### 集群状态存储
集群状态全部存在 etcd 实例中， 它提供数据高可用，以及watch 机制，数据变化能够快速通知到相关模块。

#### 调度器
当用户发送一组请求，在集群中部署一组容器，调度器模块会自动选择运行这些模块的节点。
调度器会监听未被调度的 pods，并且通过 binding的api调用，将他们与合适的节点绑定。 其中会考虑服务依赖，服务亲和度，和其他约束条件。
支持用户自定义调度。

### Worker

#### Kubelet
是 K8S 中最重要的prominent控制器，实现Pods 和 Nodes api 在容器层面的功能。K8S运行隔离的容器作为执行器，而不是传统的进程和系统包的形式， 这样不止应用容器之间可以隔离，应用和他们执行的节点也能隔离。这是解耦应用之间的管理 和 集群下物理/虚拟设备管理。
同样也关联一个 cAdvisor 资源监控 agent

#### 容器 runtime
用于抽象容器的 runtime，包括least docker (for Linux and Windows), rkt, cri-o, and frakti.

#### Kube Proxy
管理 pods 之间的网络访问策略，比如负载均衡。创建一个虚ip， 并且通过iptables 将请求重定向到正确的服务和 pods 中。Service endpoints are found primarily via DNS.


### 其他 add on 组件
* DNS
* Ingress controller
* Heapster (resource monitoring)
* Dashboard (GUI)

### 文档
https://github.com/kubernetes/community/blob/master/contributors/design-proposals/architecture/architecture.md