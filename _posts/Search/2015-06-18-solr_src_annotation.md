---
layout: post
title: "Solr Code Annotation"
keywords: ["distributed","Search","SolrCloud"]
description: "SolrCloud Leader Elect"
category: "distributed"
tags: ["distributed","Search","SolrCloud"]
---

Solr Query Syntax and Parsing 查询语法

>
1. [Common Query Parameters](https://cwiki.apache.org/confluence/display/solr/Common+Query+Parameters)
2. [The Standard Query Parser](https://cwiki.apache.org/confluence/display/solr/The+Standard+Query+Parser)
3. [The DisMax Query Parser](https://cwiki.apache.org/confluence/display/solr/The+DisMax+Query+Parser)
4. [The Extended DisMax Query Parser](https://cwiki.apache.org/confluence/display/solr/The+Extended+DisMax+Query+Parser)
5. [Function Queries](https://cwiki.apache.org/confluence/display/solr/Function+Queries)
6. [Local Parameters in Queries](https://cwiki.apache.org/confluence/display/solr/Local+Parameters+in+Queries)
7. [Other Parsers](https://cwiki.apache.org/confluence/display/solr/Other+Parsers)

查询关键流程

```
SearchHandler.handleRequestBody( c.prepare(rb)--QueryComponent.prepare())--> SearchHandler.handleRequestBody( c.process(rb)--QueryComponentprocess())-->SearchHandler.handleRequestBody( c.finishStage(rb)--QueryComponent.finishStage())
```
QueryComponent.prepare中进行参数解析，得到QParser，Query,FilterQuery等

```

```
>
1. [Solr dismax 源码详解以及使用方法](http://www.wxdl.cn/index/solr-dismax.html)
2. [Solr 的edismax与dismax比较与分析](http://www.linuxidc.com/Linux/2012-10/72373.htm)
3. [Solr 查询中fq参数的解析原理](http://blog.sina.com.cn/s/blog_56fd58ab0100v3up.html)

mm参数   

1. [Solr’s mm parameter – Explanation of Min Number Should Match](http://blog.thedigitalgroup.com/vijaym/solrs-mm-parameter-explanation-of-min-number-should-match/)   
2. [Min Number Should Match Specification Format](http://lucene.apache.org/solr/6_2_1/solr-core/org/apache/solr/util/doc-files/min-should-match.html)   
3. [Mininum Shoud Match-ES](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-minimum-should-match.html)   
4. [elasticsearch中minimum_should_match的一些理解](http://blog.csdn.net/xiao_jun_0820/article/details/51095521)

solr mm 参数例子

1，mm=a,a > 0 （clauses个数*百分数向下取整 < a）

首先依据分词得到clauses数，比如现在歌手字段（singer:简弘亦。对应ik分词结果为简，弘，亦三个） 然后搜索singer:小幸运简弘亦，此时依据ik分词为（小，幸运，简，弘，亦）类似于5个clauses，如果mm=1，mm=2或者mm=3都可以出搜索结果，是如果正数百分数，搜索词（小幸运简弘起）是5个clauses那么要想匹配上简弘亦，最终clauses个数*百分数 的结果向下去整，必须小于3,这样当搜索词（小幸运简弘起）为是5个clauses，百分数最大为59%，即mm=2小于59%，需要match的clauses语句为数5*59%小于=2  如果搜索词是“小幸运简弘”或者 “幸运简弘起” 由于ik后是四个词，也就是4个clauses，那么最大74%，即4*74%小于3=2   
### 分词（ik）与mm的一个坑（mm=100，也就是要求Query分词产生的每个clauses都必须匹配上）   
搜索词 幸福蓝海（ik分词：幸福，蓝，海）   
索引 幸福蓝海国际影城（ik分词：幸福，蓝，海国，国际，影城）   
这个为啥坑呢：   
1. 首先是mm=100这个要求高了   
2. 再次要求从索引的序列中获取的正向子序列作为query产生的分词不能超出原索引序列的分词，当然如果如果query本身超出了索引的序列也是不行的
