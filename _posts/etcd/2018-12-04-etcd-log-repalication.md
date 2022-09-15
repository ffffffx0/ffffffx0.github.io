---
layout: post
title: "Etcd log replication"
keywords: ["distributed","etcd"]
description: "Etcd log replication"
category: "distributed"
tags: ["etcd"]
---

### 什么时候Commit
当follow append log后返回leader,再次调用maybeCommit()时，第一次调用时由于follow都没有append log

半数或以上的计算是将所有的index排序后，取中间的index的大小比较？以下是过半数逻辑的优化pr: 

![maybeCommit](https://raw.githubusercontent.com/2pc/2pc.github.io/master/images/maybeCommit.jpg)

r.quorum()取半数以上

```
func (r *raft) quorum() int { return len(r.prs)/2 + 1 }
```
