---
layout:     post
title:      mysql-问题
subtitle:   
date:       2020-03-09
author:     sugar-foxs
catalog: 	true
tags:
    - mysql
---

本文主要记录一些工作中遇到的mysql相关的问题。

<!-- more -->

## 字段大小写敏感问题
- 问题是这样的：有个字符串类型字段且唯一，但是由于一个是'id',一个是'ID'。查询时本来应该是一条记录，但是出现多条。
- 这是由于字段大小写不敏感导致查询结果和预想不一致。
- 解决方法：查询时在字段前加上BINARY。select * from [表名] where BINARY [条件]='条件';



