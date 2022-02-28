---
layout:     post
title:      docker资源隔离
subtitle:   
date:       2022-02-16
author:     sugar-foxs
catalog: 	true
tags:
    - docker
---

docker如何实现资源的隔离，涉及cgroups、namespace和unionFS。
<!-- more -->

- cgroups
CGroups 全称control groups，cgroups 是 Linux 内核提供的一种机制，这种机制可以根据特定的行为，把一系列系统任务及其子任务整合（或分隔）到按资源划分等级的不同组内，从而为系统资源管理提供一个统一的框架。用来限定一个进程的资源使用，由Linux内核支持，可以限制和隔离Linux进程组 (process groups) 所使用的物理资源 ，比如cpu，内存，磁盘和网络IO，是Linux container技术的物理基础。

- cgroups 的 API 以一个伪文件系统的方式实现，即用户可以通过文件操作实现 cgroups 的组织管理。
- cgroups 的组织管理操作单元可以细粒度到线程级别，用户态代码也可以针对系统分配的资源创建和销毁 cgroups，从而实现资源再分配和管理。
- 所有资源管理的功能都以“subsystem（子系统）”的方式实现，接口统一。
- 子进程创建之初与其父进程处于同一个 cgroups 的控制组。


cpu隔离包含cpuset隔离、cpuquota隔离、cpushare隔离


内存限制：
systemd 只提供了一个参数 MemoryLimit 来对其进行控制，该参数表示某个 user 或 service 所能使用的物理内存总量。

io隔离
讨论IO资源隔离的时候，实际上有两个资源需要考虑，一个是空间，另一个是速度。
cgroup主要解决的是如何进行IO数据传输的速度限制。

- namespace
namespace用来隔离PID(进程ID),IPC,Network等系统资源。

- unionFS
