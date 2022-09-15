---
layout: post
title: "Google Spanner F1"
keywords: ["distributed"]
description: "Google Spanner F1"
category: "distributed"
tags: ["distributed","Spanner"]
---
### Google Spanner and F1 

#### Abstract

   Spanner是可扩展的、多版本、全球分布式、同步复制数据库。是数据分布在全球范围内的系统，支持外部一致性的分布式事务。

#### 1 Introduction

#### 2 Implementation
![](http://7xla7c.com1.z0.glb.clouddn.com/Spanserver)
* Universe。一个Spanner部署实例称之为一个Universe。目前全世界有3个。一个开发，一个测试，一个线上。因为一个Universe就能覆盖全球，不需要多个。
* Zones. 每个Zone相当于一个数据中心，一个Zone内部物理上必须在一起。而一个数据中心可能有多个Zone。可以在运行时添加移除Zone。一个Zone可以理解为一个BigTable部署实例。
* Universemaster: 监控这个universe里zone级别的状态信息
* Placement driver：提供跨区数据迁移时管理功能
* Zonemaster：相当于BigTable的Master。管理Spanserver上的数据。
* Location proxy：存储数据的Location信息。客户端要先访问他才知道数据在那个Spanserver上。
* Spanserver：相当于BigTable的ThunkServer。用于存储数据。

#####  2.1 SpannerServer 软件栈

![](http://7xla7c.com1.z0.glb.clouddn.com/spanner-soft-stack)

#####  2.2 Directories and Placement

#####  2.3 Data Model



#### 3 TrueTime

#### 4 ConCurrency Control

##### 4.1 TimeStamp Managerment

##### 4.2 Details 
Spanner使用TrueTime来控制并发，实现外部一致性。支持以下几种事务。
> 
* 读写事务RW
* 只读事务RO
* 快照读，客户端提供时间戳
* 快照读，客户端提供时间范围
#### 5 Evaluation 

##### 5.1 MicroBenchmarks （微基准测试）

##### 5.2 Availability（可用性）

##### 5.3 TrueTime

##### 5.4 F1

#### 6 Related Work

#### 7 Future Work




#### 论文(EN)地址：

[墙外-GoogleReseach](http://research.google.com/archive/spanner.html)

[墙内－OPEN文档](http://www.open-open.com/doc/view/824899e4892a4c8ea33ff04b6742b1d2)

中文翻译

[Google Spanner (中文版)--厦大数据库实验室](http://dblab.xmu.edu.cn/post/google-spanner/#implementation)

#### References:
[论文阅读笔记 - Spanner: Google'sGlobally-Distributed Database](http://blog.csdn.net/colorant/article/details/9126921)
