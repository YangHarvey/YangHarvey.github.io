---
title: Cloud-Native HTAP Database Architecture
date: 2024-10-17 20:45:60
tags: [cloud-native, htap, singlestore]
description: Cloud-Native HTAP Database Architecture
layout: post
---

这段时间组内对Cloud-Native HTAP数据库比较感兴趣，后续的研究工作大概率也会基于某一种Cloud-Native HTAP数据库展开。
于是这段时间我调研了一下目前主流的Cloud-Native HTAP数据库的架构。

对于Cloud-Native HTAP数据库，我觉得可以从两个角度来理解：

第一个是从HTAP的角度来理解，HTAP(Hybrid Transaction/Analytical Processing)指的是希望在一个数据库内部同时处理事务型（Transaction）和分析型（Analytical）的负载，从而省去从OLTP数据库到OLAP数据仓库的ETL（extract transform load）过程。
OLTP和OLAP是两种截然不同的负载。OLTP的单个查询通常比较短，访问数据量少，对并发性能要求比较高，OLTP数据库通常采用行存的存储引擎；OLAP的查询通常运行时间比较长，访问数据量大，对并发性能要求不高, OLAP数据库通常采用列式的存储引擎.
因此要让一个数据库同时具备良好的TP和AP性能会面临许多挑战.从存储的角度来说,如果兼顾高吞吐和低存储开销.如何设计数据同步算法.如何设计优化器(下层既有行存的执行器又有列存的向量化执行引擎,优化器的搜索空间呈指数增长).如何兼顾性能和数据新鲜度.

第二是从Cloud-Native的角度来理解,云数据库最大的特点就是资源隔离和弹性伸缩.大部分云数据库通过存算分离实现了存储层和计算层资源的隔离,同时坚持log-is-database的理念,计算层向存储层发送redo日志代替了原来的刷脏页,大大减少了网络开销,进一步提高了可扩展性.那么Cloud-Native HTAP Database在设计时也必须要考虑到可扩展性的问题.



