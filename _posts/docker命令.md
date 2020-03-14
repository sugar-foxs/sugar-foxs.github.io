---
layout:     post
title:      docker命令
subtitle:   
date:       2019-12-25
author:     sugar-foxs
catalog: 	true
tags:
    - docker
---

本文主要记录一些docker中使用到的命令。
<!-- more -->
## docker search mysql
- 搜索可用的镜像

## docker pull 
- docker pull mysql:5.6
    - 拉取mysql镜像，版本是5.6
    - 不加版本的话，默认是最新版本

## docker images
- 查看本地有哪些镜像

## docker inspect [IMAGE ID]
- 查看镜像的详情
- 如果不是自己想要的版本，可以在docker pull时指定版本号

## docker ps
- 列出所有在运行的容器信息
![hello](http://ww1.sinaimg.cn/large/dbf344a4ly1gacqxc6dxfj22iq04ytav.jpg)

- docker ps -n 5
    - 列出最近创建的5个容器信息
- docker ps -a 显示所有符合条件的，包括未运行的
- docker ps -f 根据条件过滤
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

## docker port 容器id/别名 [端口]
- 查看端口绑定情况

## docker start [容器id]
- 启动容器

## docker stop [容器id/别名]
- 停止容器

## docker restart [容器id/别名]
- 重启容器

## docker exec -it [容器id/别名] /bin/bash
- 进入容器,使用这个命令进入容器有个好处，在exit退出时不会导致容器停止。
- -i:交互式操作
- -t:终端
- 输入exit退出终端

## docker rm -f 容器id
- 删除容器

## dcoker network ls 
- 查看网络模式
- 在你创建容器的时候，不指定--network默认是bridge。


## docker mysql 修改时区
- 查看时区：docker exec [容器id/别名]  date  
- 进入容器：docker exec -it [容器id/别名] /bin/bash
- 找到通用时区文件夹： cd /usr/share/zoneinfo/Asia
- 复制当前文件夹下Shanghai：cp Shanghai /etc/localtime
- 重启容器：docker restart [容器id/别名]

## 使用redis的布隆过滤器插件
- 安装布隆过滤器：docker pull redislabs/rebloom
- 启动：docker run -p 6379:6379 --name redis-redisbloom redislabs/rebloom:latest
- 打开另一个窗口，进入启动的容器：docker exec -it redis-redisbloom bash
- 进入redis客户端：redis-cli
- 这时候便可以执行布隆过滤器的命令：
    - 增加一个值：BF.ADD newFilter foo
    - 查看是否存在：BF.EXISTS newFilter foo

## zk
- docker pull zookeeper
- docker run --privileged=true -d --name zookeeper --publish 2181:2181 zookeeper:latest
- 进入容器,name上条命令起的是zookeeper：docker exec -it zookeeper bash
- 启动zk客户端：./bin/zkClient.sh

### 本地创建zk集群
- 先创建3个挂载目录
    - mkdir zk-cluster
    - mkdir ./zk-cluster/node1
    - mkdir ./zk-cluster/node2
    - mkdir ./zk-cluster/node3
- 创建自己的bridge网络
    - docker network create --driver bridge --subnet=172.18.0.0/16 --gateway=172.18.0.1 test-net
- 创建运行容器

> docker run -d -p 2181:2181 --name    zookeeper_node1 --privileged --restart always --network test-net --ip 172.18.0.2 \  
-v /Users/guchunhui/zk-cluster/node1/volumes/data:/data \  
-v /Users/guchunhui/zk-cluster/node1/volumes/datalog:/datalog \  
-v /Users/guchunhui/zk-cluster/node1/volumes/logs:/logs \  
-e ZOO_MY_ID=1 \  
-e "ZOO_SERVERS=server.1=172.18.0.2:2888:3888;2181 server.2=172.18.0.3:2888:3888;2181 server.3=172.18.0.4:2888:3888;2181" bbebb888169c

> docker run -d -p 2182:2181 --name zookeeper_node2 --privileged --restart always --network test-net --ip 172.18.0.3 \  
-v /Users/guchunhui/zk-cluster/node2/volumes/data:/data \  
-v /Users/guchunhui/zk-cluster/node2/volumes/datalog:/datalog \  
-v /Users/guchunhui/zk-cluster/node2/volumes/logs:/logs \  
-e ZOO_MY_ID=2 \  
-e "ZOO_SERVERS=server.1=172.18.0.2:2888:3888;2181 server.2=172.18.0.3:2888:3888;2181 server.3=172.18.0.4:2888:3888;2181" bbebb888169c

> docker run -d -p 2183:2181 --name   zookeeper_node3 --privileged --restart always --network test-net --ip 172.18.0.4 \  
-v /Users/guchunhui/zk-cluster/node3/volumes/data:/data \  
-v /Users/guchunhui/zk-cluster/node3/volumes/datalog:/datalog \  
-v /Users/guchunhui/zk-cluster/node3/volumes/logs:/logs \  
-e ZOO_MY_ID=3 \  
-e "ZOO_SERVERS=server.1=172.18.0.2:2888:3888;2181 server.2=172.18.0.3:2888:3888;2181 server.3=172.18.0.4:2888:3888;2181" bbebb888169c



