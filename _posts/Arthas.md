---
layout:     post
title:      Arthas
subtitle:   
date:       2020-01-14
author:     sugar-foxs
catalog: 	true
tags:
    - java
---

本文记录一下Arthas的命令。Arthas是一个非常好的java诊断工具。在遇到线上问题时，能够在不停止应用的前提下观察应用状态。

<!-- more -->

# 下载
> curl -L https://alibaba.github.io/arthas/install.sh | sh

# 监控某个进程
> java -jar arthas-boot.jar 进程pid

# dashboard
- 当前系统的实时数据面板。包含线程、cpu、内存等内容。

# tt
- 方法执行数据的时空隧道，记录下指定方法每次调用的入参和返回信息，并能对这些不同的时间下调用进行观测。

# watch
- 方法执行数据观测。让你能方便的观察到指定方法的调用情况。能观察到的范围为：返回值、抛出异常、入参，通过编写OGNL表达式进行对应变量的查看。

# heapdump
- dump java heap, 类似jmap命令的heap dump功能。
- heapdump /Users/guchunhui/dump.hprof  (dump到指定文件)

# dump
- dump 已加载类的bytecode到特定目录。

# sc 
- 查看JVM已加载的类信息。

# sm
- 查看已加载类的方法信息。