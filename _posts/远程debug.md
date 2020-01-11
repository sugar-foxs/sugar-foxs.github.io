---
layout:     post
title:      远程debug
subtitle:   
date:       2020-01-10
author:     sugar-foxs
catalog: 	true
tags:
    - java
categories:
    - learn
---

本篇主要介绍远程debug的方法和原理。
<!-- more -->

# 远程debug的方法
![e26987facabe4fe880b3bbd5d5dca99e_039a1fb5d3b29cc76b1177e798211722.jpg](http://ww1.sinaimg.cn/large/dbf344a4ly1garr9qdj5lj21tk11gx2s.jpg)

- 在Edit Configurations中新建一个Remote
- 配置下远程服务的host和port。
- 远程服务启动时必须使用如下参数才能debug
- -agentlib:jdwp=transport=dt_socket,server=y,suspend=n,address=9806
    - jdwp:是 Java Debug Wire Protocol的缩写
    - server=y表示是监听其他debugclient端的请求
    - address=9806表示服务会在端口号9806监听debug请求，客户端必须设置这个端口号才能进行dubug
    - suspend表示是否在调试客户端建立连接之后启动VM。如果为y，那么当前的VM就是suspend直到有debug client连接进来才开始执行程序。如果你的程序不是服务器监听模式并且很快就执行完毕的，那么可以选择在y来阻塞它的启动。为n则先启动，并监听端口，等待客户端连接。
- 调试模式有两种：attach和listen。图中选的是attach。
    - attach:远程服务启动一个端口等待调试客户端去连接。
    - listen:调试客户端去监听一个端口，当调试服务端准备好了，就会进行连接。
- 传输方式：mac和linux默认使用Socket进行数据传输，windows使用共享内存方式。

# 远程debug的原理
远程服务是用来上述的参数启动后，会在指定的端口上进行监听，等待调试客户端去连接。我们在idea使用remote debug时就是启了一个客户端，连接远程服务器。两者使用debug协议（jdwp）进行通信实现远程调试。
