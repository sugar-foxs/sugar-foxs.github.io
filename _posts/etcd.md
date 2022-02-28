---
layout:     post
title:      etcd入门
subtitle:   
date:       2021-05-10
author:     sugar-foxs 
catalog: 	true
tags:
    - etcd
---

本文是关于etcd的介绍。
<!-- more -->
# etcd简介
etcd是一个可靠的分布式KV存储，其底层使用Raft算法保证一致性，主要用于共享配置和服务发现。目前提供配置共享和服务发现的组件还是比较多的，大家最为熟悉的就是zookeeper了。简单列举下etcd相较于zookeeper的优势：
- 一致性协议：etcd底层采用了raft协议，zookeeper采用了ZAB协议，ZAB协议是一种类Paxos的一致性协议。目前公认的是Raft比Paxos抑郁理解，工程化也较为容易。
- API接口：etcd v2版本提供了Http+json的调用方式，v3版本使用Grpc与服务端交互，grpc本身是跨平台的。
- 性能：etcd集群可以支持每秒10000+次的写入，性能相当可观，由于zookeeper。
- 安全性：etcd支持TLS访问，而zookeeper在权限控制这方面略显粗糙。

