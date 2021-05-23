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