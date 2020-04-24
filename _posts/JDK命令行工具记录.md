---
layout:     post
title:      JDK命令行工具记录
subtitle:   
date:       2020-04-06
author:     sugar-foxs
catalog: 	true
tags:
    - java
---

本文主要记录JDK命令行工具的使用。
<!-- more -->

# jps
- jps命令应该是工作中用的最多的命令了。用来查看服务器所有的java进程。但是有些参数你可能并不知道。
- jps -q : 只输出pid。
- jps -l : 输出主类的全名，如果进程执行的是Jar包，输出Jar路径。
- jps -v : 输出虚拟机进程启动时JVM参数。
- jps -m : 输出传递给Java进程main()函数的参数。

# jstat
- 是一个监视虚拟机各种运行状态信息的命令行工具，可以显示本地或者远程虚拟机进程中的类信息、内存、垃圾收集、JIT 编译等运行数据。

# jstack
- 用来生成虚拟机当前时刻的线程快照。可用于定位线程长时间出现停顿的原因。
- jstack pid

# jmap
- 用来生成堆转储快照。当发生OOM时，用于生成dump文件。
- jmap -dump:format=b,file=指定生成的dump文件路径 pid

# jhat
- 用于分析dump文件，它会建立一个本地http服务器，端口默认为7000，让用户可以在浏览器上查看分析结果。
- jhat dump文件路径

# jinfo
- 输出当前进程的全部参数和系统属性。
- jinfo -flag name pid :输出对应名称的参数的具体值。
- jinfo可以在不重启虚拟机的情况下，可以动态的修改jvm参数。
    - jinfo -flag [+|-]name pid :开启或者关闭对应名称的参数。
