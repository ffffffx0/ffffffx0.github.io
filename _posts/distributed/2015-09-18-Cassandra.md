---
layout: post
title: "Cassandra种种"
description: "Cassandra种种"
category: "distributed"
tags: ["distributed","Cassandra"]
---


#### 一致性等级ConsistencyLevel
[Configuring data consistency-官方文档Apache Cassandra™ 2.0](http://docs.datastax.com/en/cassandra/2.0/cassandra/dml/dml_config_consistency_c.html)

```
ANY         (0),
ONE         (1),
TWO         (2),
THREE       (3),
QUORUM      (4),
ALL         (5),
LOCAL_QUORUM(6, true),
EACH_QUORUM (7),
SERIAL      (8),
LOCAL_SERIAL(9),
LOCAL_ONE   (10, true);
```

#### 维护最终一致性
>逆熵（Anti-Entropy）
读修复（Read Repair）
提示移交（Hinted Handoff）
分布式删除
