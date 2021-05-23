---
layout:     post
title:      etcd命令
subtitle:   
date:       2020-09-06
author:     sugar-foxs 
catalog: 	true
tags:
    - etcd
---

本文主要介绍etcdctl命令。
<!-- more -->
# mac安装etcd
- brew install etcd
- 默认etcdctl APIVERSION是2，通过设置export ETCDCTL_API=3，设置APIversion为3
- 执行etcd命令启动本地etcd服务


1. 指定etcd集群
etcdctl --endpoints=$ENDPOINTS member list

2. 向etcd增加key-value
etcdctl --endpoints=$ENDPOINTS put foo "Hello World!"

3. 查询
etcdctl --endpoints=$ENDPOINTS get foo
- 输出格式可指定
etcdctl --endpoints=$ENDPOINTS --write-out="json" get foo
- 基于前缀查询，具有相同前缀的key都会被找到
etcdctl --endpoints=$ENDPOINTS get fo --prefix

4. 删除
- 删除key为foo的键值对
etcdctl --endpoints=$ENDPOINTS del foo
- 带有前缀fo的key都被删除
etcdctl --endpoints=$ENDPOINTS del fo --prefix

