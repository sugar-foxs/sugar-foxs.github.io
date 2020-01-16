---
layout:     post
title:      druid参数配置
subtitle:   
date:       2020-01-15
author:     sugar-foxs
catalog: 	true
tags:
    - druid
    - spring
---

本文介绍druid数据源的参数配置。

<!-- more -->

- spring.druid.initialSize
    - 初始化时建立物理连接的个数。初始化发生在显示调用init方法，或者第一次getConnection时。
- spring.druid.minIdle 
    - 最小连接池数量
- spring.druid.maxActive 
    - 最大连接池数量
- spring.druid.maxWait 
    - 获取连接时最大等待时间，单位毫秒。配置了maxWait之后，缺省启用公平锁，并发效率会有所下降，如果需要可以通过配置useUnfairLock属性为true使用非公平锁。
- spring.druid.testWhileIdle
    - 建议配置为true，不影响性能，并且保证安全性。申请连接的时候检测，如果空闲时间大于timeBetweenEvictionRunsMillis，执行validationQuery检测连接是否有效。
- spring.druid.timeBetweenEvictionRunsMillis
    - Destroy线程会检测连接的间隔时间，如果连接空闲时间大于等于minEvictableIdleTimeMillis则关闭物理连接。
    - testWhileIdle的判断依据，详细看testWhileIdle属性的说明
- spring.druid.minEvictableIdleTimeMillis
    - 连接保持空闲而不被驱逐的最小时间
- spring.druid.validationQuery
    - 用来检测连接是否有效的sql，要求是一个查询语句，常用SELECT 1 FROM DUAL。如果validationQuery为null，testOnBorrow、testOnReturn、testWhileIdle都不会起作用。
- spring.druid.testOnBorrow
    - 申请连接时执行validationQuery检测连接是否有效，做了这个配置会降低性能。
- spring.druid.testOnReturn
    - 归还连接时执行validationQuery检测连接是否有效，做了这个配置会降低性能。
- spring.druid.poolPreparedStatements
    - 是否缓存preparedStatement，也就是PSCache。PSCache对支持游标的数据库性能提升巨大，比如说oracle。在mysql下建议关闭。
- spring.druid.maxPoolPreparedStatementPerConnectionSize
    - 要启用PSCache，必须配置大于0，当大于0时，poolPreparedStatements自动触发修改为true。在Druid中，不会存在Oracle下PSCache占用内存过多的问题，可以把这个数值配置大一些，比如说100
- spring.druid.filters=stat,wall,log4j
    - 属性类型是字符串，通过别名的方式配置扩展插件，常用的插件有：
    - 监控统计用的filter:stat
    - 日志用的filter:log4j
    - 防御sql注入的filter:wall
- spring.druid.connectionProperties=druid.stat.mergeSql=true;druid.stat.slowSqlMillis=5000