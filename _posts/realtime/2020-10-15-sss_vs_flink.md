---
title: Structured Streaming VS Flink
tagline: ""
category : Java
layout: post
tags : [Java, spark, flink]
---

再记录下spark以及flink的一些差异

### 实时处理方面

1. spark 这里只关注structured streaming,因为工作中以sss为主，sss通常还是micro batch， 默认100 milliseconds，    
但是在since Spark 2.3引入新的Continuous Processing，这个老实说感觉挺鸡勒的

2. sss在多流join方面也是有些限制的,[Support matrix for joins in streaming queries](https://spark.apache.org/docs/latest/structured-streaming-programming-guide.html#support-matrix-for-joins-in-streaming-queries)

3. 贴下官方的描述[Continuous Processing ](https://spark.apache.org/docs/latest/structured-streaming-programming-guide.html#continuous-processing)   
Internally, by default, Structured Streaming queries are processed using a micro-batch processing engine, which processes data streams as a  
series of small batch jobs thereby achieving end-to-end latencies as low as 100 milliseconds and exactly-once fault-tolerance guarantees. However,   
since Spark 2.3, we have introduced a new low-latency processing mode called Continuous Processing, which can achieve end-to-end latencies   
as low as 1 millisecond with at-least-once guarantees. Without changing the Dataset/DataFrame operations in your queries, you will be able to   
choose the mode based on your application requirements   
4. 另外： 对的支持也是相当有限的，即使在2.4版本   
![Supported Queries](https://raw.githubusercontent.com/2pc/2pc.github.io/master/images/2.4.png)   


### 批处理

#### spark 

1. spark目前在离线批处理方面应该比flink应用的更加广泛了，即便是用的hive引擎页大多是spark   
2. flink已经整合了hive,当然也在整合delta lake, hudi， iceberg等的，具备流批处理的能力
