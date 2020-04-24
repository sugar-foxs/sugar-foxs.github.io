---
layout:     post
title:      linux命令记录
subtitle:   
date:       2020-02-8
author:     sugar-foxs
catalog: 	true
tags:
    - linux
---

本文是对于linux命令的记录。

<!-- more -->

# 查看文件夹大小
- du -h --max-depth=1 
    - h:以K，M，G为单位，提高信息的可读性，不加h默认是以byte为单位
    - 目录层数限制为1

# 查看当前目录各文件大小
- ls -lh 
    - 以K，M，G为单位

# 显示目前在Linux系统上的文件系统的磁盘使用情况统计
- df -h

# 计算文件字数命令
- wc filename ：将返回文件的行数、字数，以及字节数。

# 不同后缀文件的解压缩命令
## tar
- 解压：tar -xzvf xxx.tar.gz
- 压缩：tar –czvf xxx.tar.gz *
> -z:对应gz  
-x: 解压 --extract的简写，从备份中还原文件  
-c: 压缩 --create的简写，创建新的备份文件  
-v：显示所有过程
-f: 使用档案名字，切记，这个参数是最后一个参数，后面只能接档案名


