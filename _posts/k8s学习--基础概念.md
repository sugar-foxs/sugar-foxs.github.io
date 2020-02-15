---
layout:     post
title:      k8s学习--基础概念
subtitle:   
date:       2020-02-8
author:     sugar-foxs
catalog: 	true
tags:
    - k8s
---

本文记录一些对于k8s的学习。
<!-- more -->

## k8s的基本知识
### Service
- 在k8s中，Service(服务)是分布式集群架构的核心。一个service对象拥有如下特征：
    - 拥有一个唯一指定的名字
    - 拥有一个唯一的虚拟ip和端口
    - 能够提供某种远程服务能力
    - 被映射到了提供这种服务能力的一组容器应用上
- Service的服务进程目前都基于Socket,Service本身一旦创建就不再变化。

### Pod
- pod运行在一个称之为节点（Node）的环境中，node可以是物理机，也可以是虚拟机。通常一个节点可以运行上百个pod。
- 将每个服务进程包装到pod中，使其成为pod中运行的一个容器。
- 通过标签建立Service和pod之间的关联关系。为pod设置name=xxx的标签，Service设置标签选择器，选择条件为name=xxx,这样service会作用于具有name=xxx标签的所有pod。
- 每个pod中都有一个Pause容器，pause容器的状态代表了整个pod的状态。

### Master和Node
- k8s集群包含一个Master节点和多个工作节点Node。
#### Master
- 在Master节点上运行了集群管理相关的进程：kube-apiserver,kube-controller-manager,kube-schedule。这些进程实现了整个集群的资源管理，pod调度，弹性伸缩，安全控制，系统监控和纠错等管理功能。k8s的所有控制命令都是发给Master，由Master负责具体的执行过程。
- kube-apiserver：提供了HTTP Rest接口的廉服务进程，是k8s里所有资源增、删、改、查等操作的唯一入口。是集群控制的入口进程。
- kube-controller-manager：k8s里所有资源对象的自动化控制中心。
- kube-schedule：负责资源调度（pod调度）的进程。
- etcd server：Master节点往往还启动了一个etcd Server进程，k8s所有资源对象的数据全部保存在etcd中。

#### Node
- Node节点运行真正的应用程序，Node上运行着k8s的kubelet,kube-proxy进程，这些进程负责pod的创建，启动，监控，重启，销毁以及实现软件模式负载均衡器。还有docker引擎，负责本机容器的创建和管理工作。
- kubelet:默认会向Master注册自己。注册完成，kubelet会定期向master汇报自身的情报，例如：操作系统、docker版本、机器的cpu和内存情况、以及有哪些pod在运行，这样master便可以知道每个node的资源使用情况，并实现高效均衡的资源调度策略。

### label标签
- label是一个key=value的键值对，key和value都由用户指定。
- label使得被管理对象能够被分组管理，同时实现整个群的高可用性。

### Replication Controller（RC）
- RC能够解决服务扩容和服务升级的问题。
- 创建一个RC定义文件，包含以下内容：目标pod的定义、目标pod需要运行的副本数量、要监控的目标pod的标签。
- 创建好RC定义文件并提交到k8s集群后，Master节点上的ontroller Manager组件就会得到通知，通过RC中定义的标签筛选出对应的pod实例并实时监控其状态和数量，如果实例数量少于定义的副本数量，则根据RC中定义的pod模板创建一个新的pod,然后将pod调度到何时的Node上运行，直到pod数量达到设置的副本数量。
- kubectl提供了top和delete命令来一次性删除RC和RC控制的所有pod。

### Replica Set
- 是RC的升级版，唯一的区别是：支持基于集合的label selector。

### Deployment
- Deployment相对于RC的升级是：可以随时知道pod的部署进度。

### Horizontal Pod AutoScaler (HPA)
- pod横向扩容，和RC、Deployment一样，也属一种k8s的资源对象。
- 通过追踪分析RC控制的所有Pod的负载变化情况，来确定是否需要针对性调整目标Pod的副本数。

### k8s中的3种ip
- Node IP :k8s集群中每个节点的物理网卡的IP地址，

- Pod IP：每个Pod的IP地址，它是Docker Engine 根据docker0网桥的IP地址段进行分配的，通常是一个虚拟的二层网络，不同Node上的Pod能够进行通信，就是通过Pod IP所在的虚拟二层网络进行通信的，而真实的TCP/IP流量则是通过Node IP所在的物理网卡流出的。

- Cluster IP：
    - 仅仅作用Kubernetes Service这个对象，并由k8s管理和分配ip地址。
    - 无法被ping
    - 只能结合Service Port组成一个具体的通信端口，单独的Cluster IP不具备TCP/IP 通信的基础，并且它们属于k8s集群这样一个封闭的空间，集群之外的节点如果需要访问这个通信端口，则需要做一些额外的工作。

### Volume(存储卷)
- 是pod中能被多个容器访问的共享目录。

### Namespace
- 用于实现多租户的资源隔离，k8s集群启动后默认会新建一个名为"default"的Namespace
- kubectl get pods --namespace=development 用来查看命名空间是development的对象。

### Annotation(注解)
- Annotation和Label类似，也是用key/valuef方式进行定义。用来标记资源对象的一些特殊信息：build信息、release信息、Docker镜像信息、日志库等资源库的地址信息、程序调试工具信息、团队的联系信息。


> 摘自《Kuberetes权威指南》


