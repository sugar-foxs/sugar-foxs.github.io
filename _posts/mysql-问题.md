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

## 没用到索引的情况
- 查询中对索引字段有函数操作
- 隐式类型转换，这里其实也是有函数操作，因为优化器会对需要转换的加cast
- 隐式字符编码转换，这里其实也是有函数操作，因为优化器会对需要编码转换的加convert

- 模糊搜索，因为不能根据索引的有序性去查询。

## count
- 性能：count(字段) < count(主键 id) < count(1) ≈ count(*)



