---
title: Kafka Raft& Kafka3 源码剖析合集
tagline: ""
category : kafka
layout: post
tags : [kafka]
---

[Kafka3 kraft源码剖析合集 ](https://github.com/2pc/notes/issues?q=is%3Aissue+is%3Aopen+label%3Akafka)   
. [Kafka3.x 请求处理Apis](https://github.com/2pc/notes/issues/13)   
. [Kafka3.x 元数据MetadataCache](https://github.com/2pc/notes/issues/12)   
. [Kafka 3.x Kraft 实现源码分析](https://github.com/2pc/notes/issues/10)   
. [Kafka如何保证消息的可靠性](https://github.com/2pc/notes/issues/8)   

[kafka 0.9 0.10 源码剖析合集](https://github.com/2pc/notes/tree/master/mq/kafka)

选举时序图1
![启动](https://raw.githubusercontent.com/2pc/2pc.github.io/master/_posts/images/kafka_raft1.png)

选举时序图2 raft loop
![启动](https://raw.githubusercontent.com/2pc/2pc.github.io/master/_posts/images/kafka_raft2.png)


[KIP-595: A Raft Protocol for the Metadata Quorum](https://cwiki.apache.org/confluence/display/KAFKA/KIP-595%3A+A+Raft+Protocol+for+the+Metadata+Quorum)   
[KAFKA-10492; Core Kafka Raft Implementation (KIP-595) #9130](https://github.com/apache/kafka/pull/9130)   

