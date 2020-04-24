---
layout:     post
title:      分布式session
subtitle:   
date:       2020-03-12
author:     sugar-foxs
catalog: 	true
tags: 分布式
---

分布式session的实现方式：
1. session复制
- 应用服务器开启web容器的session复制功能，在集群中的几台服务器之间同步session对象，使得每台服务器上都保存所有的session信息，这样任何一台宕机都不会导致session的数据丢失，服务器使用session时，直接从本地获取。
- 缺点：这种方式在应用集群达到数千台的时候，就会出现瓶颈，每台都需要备份session，出现内存不够用的情况。
<!-- more -->
2. session绑定
- 利用hash算法，比如nginx的ip_hash,使得同一个ip的请求分发到同一台服务器上。
- 缺点：一旦某台服务器宕机，该机器上的session就不存在了，用户请求切换到其他机器后没有session，无法完成业务处理。

3. 利用cookie记录session
- 缺点是：受cookie大小的限制，能记录的信息有限；每次请求响应都需要传递cookie，影响性能，如果用户关闭cookie，访问就不正常。

4. session服务器
- 利用独立部署的session服务器（集群）统一管理session，服务器每次读写session时，都访问session服务器。
- 利用分布式缓存（redis), 数据库等统一存储session。
- 构建单点登录系统（sso)