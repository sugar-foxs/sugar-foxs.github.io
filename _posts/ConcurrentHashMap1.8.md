---
layout:     post
title:      ConcurrentHashMap1.8原理解读
subtitle:   
date:       2020-04-06
author:     sugar-foxs
catalog: 	true
tags:
    - java
---

本文通过对ConcurrentHashMap1.8版本的代码进行解读来了解其原理。
<!-- more -->
# ConcurrentHashMap
- 待写... 可先参考[HashMap? ConcurrentHashMap? 相信看完这篇没人能难住你！](https://crossoverjie.top/2018/07/23/java-senior/ConcurrentHashMap/) ,但是该篇文章中关于ConcurrentHashMap1.8死循环的内容持怀疑态度。
## put
## get

## size
- 今天在学习LongAdder的时候，突然发现ConcurrentHashMap中获取size的方法和LongAdder获取值的方法竟然是一样的。OMG... 详情可参考{% post_link java并发-LongAdder %}