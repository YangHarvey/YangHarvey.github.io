---
title: 内存分离的可扩展范围索引
date: 2024-05-26 00:00:00
tags: [memory disaggregated, index]
description: 内存分离的可扩展范围索引
layout: post
---

这里我们分享在VLDB2024上的一篇的论文[DEX](https://arxiv.org/abs/2405.14502)（这里是`arxiv`上的扩展版本）

# 内存分离的可扩展范围索引

## 1. 研究背景

​	内存分离可以使许多针对内存优化的范围索引（例如：B+树）能够突破一台机器的限制，扩展到多台机器中上，增加硬件的使用率并降低成本。但是内存分离并不等于可扩展，要实现范围索引在内存层的可扩展还需要存在以下几个问题：

- **基础缓存(本文中缓存指`DRAM`)：**如果计算层有本地的`DRAM`，那么这部分DRAM需要新的缓存策略。过去的缓存策略基于传统磁盘数据库的假设，磁盘的读写速度和本地DRAM相差1000倍左右，然后远端内存层（基于RDMA）的访问速度和本地DRAM只相差10倍左右。因此这种情况下，如何维护DRAM之间的缓存一致性以及同步机制变得更加重要。而且，我们不应该假设计算层一定有本地DRAM，目前工业界还没有将内存分离架构应用到实践中，因此如果计算层的本地DRAM被严重限制，那么通用的缓存策略对于树形结构的B+树索引来说性能可能不会表现得很好
- **无原则的计算卸载：**计算卸载是指将部分计算任务卸载到内存节点中，从而减少RDMA通信的开销。举个例子，假设我们需要做一个索引操作，取B+树叶子节点中的某条数据。如果没有计算卸载，那么我们要从B+树根节点开始依次取页面直到找到对应的叶子节点，这个过程需要多次的RDMA通信开销。而通过计算卸载，我们仅仅向内存节点发送请求要求做索引查找操作，由内存节点做计算任务并将查找到的数据返回给计算层，这个过程会减少很多通信开销。然而，内存节点的计算能力有限，并不能承载过多的计算任务，因此需要选择什么样的任务需要卸载到内存层？什么时候卸载？卸载多少？（what, when, how much to offload）
- **数据不一致性：**内存层的引入带来了新的数据不一致的挑战。即使不考虑计算层的缓存和计算卸载，也需要同步机制（锁）保证数据不会被多个计算线程同时修改，不幸的是目前基于RDMA的锁机制代价很高。如果考虑计算层的缓存，那么保证多个计算节点的DRAM一致性也是必要的。进一步，如果考虑计算卸载，当内存节点执行计算任务之前，必须要保证当前内存中B+树处于一致性的状态，在执行计算任务时，也需要通过锁机制或者一致性信息保证B+树的一致性。当计算任务结束，内存节点需要更新计算层缓存中的旧数据。

## 2. DEX

文章提出了DEX，一种在内存分离环境下可扩展的B+树索引。DEX提出了一些新技术和一些现有技术有效减轻了上述的一些问题**。**

- **计算层的逻辑分区：**逻辑分区减少了

