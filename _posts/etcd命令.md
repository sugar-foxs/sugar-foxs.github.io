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

# etcdctl基本使用
可分为数据库操作和非数据库操作。数据库操作其实就是对数据的增删改查；非数据操作包含对集群信息的操作、监听等等。
## 数据库操作
### put 指定某个键的值
- etcdctl --endpoints=http://127.0.0.1:2379 put /testdir/foo 'Hello World'

### get 查询某个键的值
- etcdctl --endpoints=http://127.0.0.1:2379 get /testdir/foo
- 输出格式可指定
etcdctl --endpoints=http://127.0.0.1:2379 get /testdir/foo --write-out="json"
- 基于前缀查询，具有相同前缀的key都会被找到
etcdctl --endpoints=http://127.0.0.1:2379 get /testdir/fo --prefix

### del 删除某个键值对
- 删除key为foo的键值对
etcdctl --endpoints=http://127.0.0.1:2379 del /testdir/foo
- 带有前缀fo的key都被删除
etcdctl --endpoints=http://127.0.0.1:2379 del /testdir/fo --prefix

### 租约
- 1.创建租约
- etcdctl --endpoints=http://127.0.0.1:2379 lease grant 1800
- 返回值中包含租约id

- 2.查看所有租约
- etcdctl --endpoints=http://127.0.0.1:2379 lease list

- 3.绑定租约
- etcdctl --endpoints=http://127.0.0.1:2379 put /testdir/fo 'ffff' --lease ${leaseid}

- 查看某个租约剩余时间
- etcdctl --endpoints=http://127.0.0.1:2379 lease timetolive ${leaseid}
- 查看某个租约绑定的key
- etcdctl --endpoints=http://127.0.0.1:2379 lease timetolive ${leaseid} --keys

- 4.删除租约
- 先查看下绑定的key
- 删除租约 etcdctl --endpoints=http://127.0.0.1:2379 lease revoke ${leaseid}
- 删除租约会，绑定的key也被删除

- 5.续租
- etcdctl --endpoints=http://127.0.0.1:2379 lease keep-alive ${leaseid}


## 非数据库操作
- 查看集群健康状态
- etcdctl --endpoints=http://127.0.0.1:2379 endpoint health

- 查看集群status
- etcdctl --endpoints=http://127.0.0.1:2379 endpoint status
- endpoint, ID, version, db size, is leader, raft term, raft index

- 查看etcd集群
etcdctl --endpoints=http://127.0.0.1:2379 member list

- 