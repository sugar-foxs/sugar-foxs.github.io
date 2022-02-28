---
layout:     post
title:      go入门
subtitle:   
date:       2021-2-8
author:     sugar-foxs
catalog: 	true
tags:
    - go
---

本文是学习go语言过程中的记录。

<!-- more -->

# 环境设置
版本13之前使用export k=v 例如export GOPROXY=https://goproxy.io ,改.bash_profile更好，因为export只对当前shell有用。
之后使用go env w k=v 例如go env w GOPROXY=https://goproxy.io

# mac go版本升级
1. 删除 /usr/local/go 目录和/usr/local/Cellar/go 目录
2. 从这个地址 http://mirrors.ustc.edu.cn/golang/ 下载合适的mac版本pkg进行安装
3. 重启mac
4. go version查看是否安装成功





