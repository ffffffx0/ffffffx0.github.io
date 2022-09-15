---
title: RocketMQ事务性实现
tagline: ""
category : rocketmq
layout: post
tags : [rocketmq]
---

默认最大次数transactionCheckMax=15，以及间隔时间transactionCheckInterval=60*1000
```
    /**
     * The maximum number of times the message was checked, if exceed this value, this message will be discarded.
     */
    @ImportantField
    private int transactionCheckMax = 15;

    /**
     * Transaction message check interval.
     */
    @ImportantField
    private long transactionCheckInterval = 60 * 1000;
```
