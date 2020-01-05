---
layout:     post
title:      docker命令
subtitle:   
date:       2019-12-25
author:     sugar-foxs
header-img: img/home-bg-o.jpg
catalog: 	true
tags:
    - docker
---

## docker pull 
- docker pull mysql:5.6
    - 拉取mysql镜像，版本是5.6
<!-- more -->
## docker ps
- 列出所有在运行的容器信息
![hello](http://ww1.sinaimg.cn/large/dbf344a4ly1gacqxc6dxfj22iq04ytav.jpg)

- docker ps -n 5
    - 列出最近创建的5个容器信息

- docker ps -f 根据条件过滤,默认是正在运行的，加上-a会显示所有符合条件单的，包括未运行的。
    - docker ps -f ancestor=mysql  根据镜像过滤
    - docker ps -f status=running  根据状态过滤
    - docker ps -f name=mysql 根据名称过滤
    - docker ps -f label=a 根据标签过滤
    - docker ps -f before=容器id 根据启动顺序过滤
 
## docker run
- docker run -p 3316:3306 --name mysql -e MYSQL_ROOT_PASSWORD=123456 -d mysql:5.6
    - --name：为容器指定名称
    - -p：指定端口映射，格式为（主机端口:容器端口）
    - -e：设置环境变量
    - -d：后台运行容器，并返回容器ID

## docker start
- 启动容器

## docker stop
- 停止容器

## docker restart
- 重启容器

## docker mysql 修改时区
- 查看时区：docker exec [容器id/别名]  date  
- 进入容器：docker exec -it [容器id/别名] /bin/bash
- 找到通用时区文件夹： cd /usr/share/zoneinfo/Asia
- 复制当前文件夹下Shanghai：cp Shanghai /etc/localtime
- 重启容器：docker restart [容器id/别名]



