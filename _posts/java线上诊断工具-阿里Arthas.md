---
layout:     post
title:      java线上诊断工具-阿里Arthas
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

# 启动
> java -jar arthas-boot.jar 进程pid

# 所有命令
## dashboard
- 当前系统的实时数据面板。包含线程、cpu、内存等内容。

## thread
- thread 1 | grep 'main'
- 打印线程ID 1的栈, grep出自己想看的内容。

## heapdump
- dump java heap, 类似jmap命令的heap dump功能。
- heapdump /Users/guchunhui/dump.hprof  (dump到指定文件)

## jvm
- 查看当前 JVM 的信息

## sysenv 
- 查看jvm的环境变量

## sysprop
- 查看或者修改jvm的系统属性
示例：
java.version=1.8.0_191
java.runtime.version=1.8.0_191-b12

## vmoption
- 查看和修改JVM里诊断相关的option

## getstatic
- 查看类的静态属性


## jad
- 反编译class
- jad 类路径

## mc
- 编译.java文件生成.class

## tt
- 方法执行数据的时空隧道，记录下指定方法每次调用的入参和返回信息，并能对这些不同的时间下调用进行观测。

## watch
- 方法执行数据观测。让你能方便的观察到指定方法的调用情况。能观察到的范围为：返回值、抛出异常、入参，通过编写OGNL表达式进行对应变量的查看。
- watch class-pattern method-pattern express condition-express
- express是观察表达式，观察表达式的构成主要由 ognl 表达式组成，所以你可以这样写"{params,returnObj}"，只要是一个合法的 ognl 表达式，都能被正常支持。
- watch 命令定义了4个观察事件点，即 -b 方法调用前，-e 方法异常后，-s 方法返回后，-f 方法结束后。
- 4个观察事件点 -b、-e、-s 默认关闭，-f 默认打开，当指定观察点被打开后，在相应事件点会对观察表达式进行求值并输出。
- 观察异常信息例子：
    - watch 类名 方法 "{params[0],throwExp}" -e -x 2 -n 5
    - -e 表示抛出异常才触发，throwExp表示异常信息
    - -x 表示遍历深度。
    - -n 只执行指定次数。


## trace
- trace class-pattern method-pattern condition-express
- condition-express：条件表达式是用来过滤的。
- trace demo.Main main '#cost > 10'    只展示Main类main方法调用耗时超过10ms的调用路径。

## stack
- 输出当前方法被调用的调用路径。
- stack class-pattern method-pattern condition-express

## ognl
- ognl表达式官方指南：https://commons.apache.org/proper/commons-ognl/language-guide.html
- 

## dump
- dump 已加载类的bytecode到特定目录。

## sc 
- 查看JVM已加载的类信息。

## sm
- 查看已加载类的方法信息。