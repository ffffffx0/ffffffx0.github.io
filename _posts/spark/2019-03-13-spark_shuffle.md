---
title: Spark shuffle
tagline: "spark"
category : spark
layout: post
tags : [Spark thrift]
---

### Hash Shuffle V1

 总文件数 M*R

### Hash Shuffle V2
M(Executor)*R

500个map  task 分配10个Executor,每个Executor一个core,每个Executor分配50个task, 则总文件数10*R

### Sort Shuffle V1

### Sort Shuffle V2
