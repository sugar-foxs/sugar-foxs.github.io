---
layout:     post
title:      kafka
subtitle:   
date:       2020-04-24
author:     sugar-foxs
catalog: 	true
tags:
    - kafka
---

本文是关于kafka的原理学习。
<!-- more -->
# kafka

## producer

## consumer

## 压缩

## 投票机制
- Kafka 动态维护了一个同步状态的备份的集合，简称ISR ，在这个集合中的节点都是和leader保持高度一致的，只有这个集合的成员才有资格被选举为 leader，一条消息必须被这个集合所有节点读取并追加到日志中了，这条消息才能视为提交。这个ISR集合发生变化会在ZooKeeper持久化，正因为如此，这个集合中的任何一个节点都有资格被选为leader 。这对于Kafka使用模型中，有很多分区和并确保主从关系是很重要的。因为ISR模型和f+1副本，一个Kafka topic冗余f个节点故障而不会丢失任何已经提交的消息。

# kafka管理
## 集群管理
1. 启动kafka

bin/kafka-server-start.sh -daemon <path>/server.properties

2. 关闭kafka

bin/kafka-server-stop.sh

这个脚本会搜寻机器上所有broker进程，并全部关闭。不建议使用kill -9 方式关闭进程，会导致broker状态的不一致。

另一种方法：

找到要kill的broker进程的pid之后执行 kill -s TERM $pid

3. 增加broker

启动新的broker之后，需要手动执行分区重分配操作，否则新增的broker不会自动被分配任何已有的topic分区

4. 升级broker版本

更新broker间通信版本和消息版本，这一步向所有broker的server.properties中增加下面两行。
inter.broker.protocol.version=当前版本
log.message.format.version=当前版本
依次更新代码，重启所有broker
下载新版本的kafka二进制包，放在哪自己定，只要确保使用目标版本的二进制程序即可。
再次更新broker间通信版本和消息版本
再次依次重启broker
## topic管理
1. 创建topic

创建topic的途径有如下4种：

- 执行kafka-topics.sh脚本
    - 创建test-topic，6个分区，3个副本，topic日志留存时间是3天
    - bin/kafka-topics --create --zookeeper localhost:2181 --partitions 6 --replication-factor 3 --topic test-topic --config delete.retention.ms=259200000
    - 创建test-topic2,4个分区，2个副本，手动分配
    - bin/kafka-topics --create --zookeeper localhost:2181 --topic test-topic2 --repica-assignment 0:1,1:2,0:2,1:2
- 发送CreateTopicsRequest请求
- 发送MetadataRequest请求且broker端设置了auto.create.topics.enable为true
- 向zk的/brokers/topics路径下写入以topic名称命名的子节点


2. 删除topic

删除topic的方法：

- 使用kafka-topics脚本
- 构造DeleteTopicsRequest请求
- 向zk的/admin/delete_topics下写入子节点，不推荐。
- 无论使用哪种方法，必需保证delete.topic.enable=true,否则kafka不会删除topic。

- bin/kafka-topics --delete --zookeeper localhost:2181 --topic test-topic

3. 查询topic

- bin/kafka-topics.sh --zookeeper localhost:2181 --list

4. 查询topic详情

- 查询test-topic详情，不指定topic，则查询集群所有topic
- bin/kafka-topics --zookeeper localhost:2181 --describe --topic test-topic

5. 修改topic

- 增加分区
- test-topic的分区数改为10，根据key确定分区的时候，增加分区需要考虑清楚。
- bin/kafka-topics --alter --zookeeper localhost:2181 --partitions 10 --topic test-topic
- 设置topic级别参数
- 为test-topic设置参数cleanup.policy=compact
- bin/kafka-configs.sh --zookeeper localhost:2181 --alter --entity-type topics --entity-name test-topic --add-config cleanup.policy=compact