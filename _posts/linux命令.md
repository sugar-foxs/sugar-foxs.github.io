---
layout:     post
title:      linux命令入门
subtitle:   
date:       2020-02-8
author:     sugar-foxs
catalog: 	true
tags:
    - linux
---

本文是对于linux命令的学习。

<!-- more -->

# 查看文件夹大小
- du -h --max-depth=1 
    - h:以K，M，G为单位，提高信息的可读性，不加h默认是以byte为单位
    - 目录层数限制为1

# 查看当前目录各文件大小
- ls -lh 
    - 以K，M，G为单位

# 显示目前在Linux系统上的文件系统的磁盘使用情况统计
- df :显示文件系统的磁盘使用情况统计
- df -h ：-h选项，通过它可以产生可读的格式df命令的输出

# 计算文件字数命令
- wc filename ：将返回文件的行数、字数，以及字节数。
- wc -l : 只显示行数
- wc -w : 只显示字数
- wc -c : 只显示Byte数

# 不同后缀文件的解压缩命令
## tar
- 解压：tar -xzvf xxx.tar.gz
- 压缩：tar –czvf xxx.tar.gz *
> -z:对应gz  
-x: 解压 --extract的简写，从备份中还原文件  
-c: 压缩 --create的简写，创建新的备份文件  
-v：显示所有过程
-f: 使用档案名字，切记，这个参数是最后一个参数，后面只能接档案名

# . ./a.sh
- . ./a.sh  第一个.代表source,所以相当于source ./a.sh

# tr
- tr [OPTION]... SET1 [SET2]
    - OPTION:-s,-t,-d,-c

## tr -s 替换重复的字符
1. echo "aaabbbaacccfdc" | tr -s [abcdf]
    - 返回abacfdc
2. 删除空白行
- tr -s ["\n"]
- ![1599902157734.jpg](http://ww1.sinaimg.cn/large/dbf344a4gy1ginzjdyhlwj212c0eumz7.jpg)

## tr -d 删除字符
- echo "a13HJ14fdaADff" | tr -d "[a-z][A-Z]"
    - 返回1314

## tr -t 字符替换
- echo "a1314fdasf" | tr -t [afd] [AFD]
    - 返回A1314FDsF

- 实现大小写转换
    - echo "Hello World I Love You" |tr -t [a-z] [A-Z]
    - echo "HELLO WORLD I LOVE YOU" |tr -t [A-Z] [a-z]
    - 使用字符集转换
    - echo "Hello World I Love You" |tr -t [:lower:] [:upper:]
    - echo "HELLO WORLD I LOVE YOU" |tr -t [:upper:] [:lower:]

- 字符集


## tr -c 字符补集替换
- tr -c "[a-z][A-Z]" "#"
    - 除大小字母以外的所有的字符都替换为#


