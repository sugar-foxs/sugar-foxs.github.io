---
layout:     post
title:      java-阻塞队列
subtitle:   
date:       2020-01-12
author:     sugar-foxs
catalog: 	true
tags:
    - java
    - 并发编程
---

本文主要介绍有哪几种阻塞队列，具体各个队列的原理在其他文章介绍。

<!-- more -->
> 因为是阻塞队列，所以都会使用到锁，他们都使用的是ReentrantLock。阻塞队列在线程池中用的比较多。

# ArrayBlockingQueue
> 一个使用数组实现的有界阻塞队列。在ArrayBlockingQueue内部，维护了一个定长数组，以便缓存队列中的数据对象。此队列按 FIFO（先进先出）原则对元素进行排序。创建其对象必须明确大小，像数组一样。默认采用非公平锁。见{% post_link 阻塞队列-ArrayBlockingQueue %}

# LinkedBlockQueue
> 一个使用链表实现的阻塞队列，按FIFO顺序排序。队列大小不指定的话，最大值是Integer.MAX_VALUE。

# DelayQueue
> 延迟阻塞队列。内部使用优先级队列PriorityQueue，按延迟时间delay排序。和{% post_link 延迟队列DelayedWorkQueue(ScheduledThreadPoolExecutor专用) %}类似，都是BlockingQueue的子类，都是延迟阻塞队列。

# PriorityBlockingQueue
> 使用数组实现的优先级阻塞队列，按优先级排序。

# SynchronousQueue
> 一种无缓冲的等待队列。该队列没有缓冲，生产者和消费者类似于电影中黑帮交易，必须一手交钱，一手交货，直接交易，没有中间商赚差价。