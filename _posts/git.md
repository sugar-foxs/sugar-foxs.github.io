---
layout:     post
title:      git
subtitle:   
date:       2020-03-14
author:     sugar-foxs
catalog: 	true
tags:
    - git
---

本文主要记录一些不常用的git命令。
<!-- more -->

# Git 命令
##  git blame
- 查看文件中每行的author等信息
- git blame [filename]
- 如果你用的是idea,右击编辑框左边栏，可以选择annotate,显示作者。

## git cherry-pick [commit id]
- 选择某一个分支中的一个或几个commit应用到另一个分支上。
- 假设需要将a分支的commit（commit id假设为A）应用到分支B上，你需要在a分支上使用git log拿到commit id，然后checkout到b分支上，使用cherry-pick到b分支，如果有冲突，解决掉就行。

## git tag
- 打标签是每次发布版本时都需要做的事
- git tag <name> 
    - 示例： git tag v1.0 ，表示标签名是v1.0,看到标签能够很清楚知道当前commit之前是哪个版本的。
- git tag <name> <commit id>
    - 上一个命令默认是对本地最新的commit打标签，如果需要对历史指定commit打标签，需要使用命令
- git tag -a <name> -m <message> <commit id>
    - 创建带有说明的标签，用-a指定标签名，-m指定说明文字,可以写此次版本增加了哪些功能
- git tag 可以查看所有标签
- git show <tagname> 看出某个标签的具体信息
- git push --tags
    - 将本地的所有tags推到远程

## git reflog
- 可以查看所有分支的所有操作记录（包括已经被删除的 commit 记录和 reset 的操作）
- git reset --hard commitId