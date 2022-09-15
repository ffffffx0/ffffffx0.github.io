---
layout: post
title: "Lucene评分机制(Score)"
keywords: ["Lucene","Search"]
description: "Lucene Score"
category: "Lucene"
tags: ["Lucene","Search"]
---
Lucene Score 评分机制

先来看几个数学公式，参考官方文档对[TFIDFSimilarity](http://lucene.apache.org/core/5_3_0/core/org/apache/lucene/search/similarities/TFIDFSimilarity.html)的说明

### 分值算法公式：

$$score(q,d)=coord(q,d)*queryNorm(q)\sum_{t\ in\ d }(tf( t\ in\ d )*idf(t)^2*t.getBoost()*norm(t,d)$$

其中

* tf(t in d) 表示词频，Term在当前文档中出现的次数
* idf(t) 词的逆词频
* coord(q,d) 评分因子。越多的查询项(Term)在一个文档中，说明这些文档的匹配程序越高
* queryNorm(q) 标准化因子，用于查询时比较
* t.getBoost() 权重，这个可以在查询的时候给定（如:lucene^2），也可以在索引的时候由setBoost()给定
* norm(t,d)   字段加权(Field boost ), 文档加权(Document boost)和长度因子由字段内的 Token 的个数来计算此值，字段越短，评分越高，在做索引的时候由 Similarity.lengthNorm 计算,norm这个值在索引的时候会encode 成一个byte 保存在索引中，搜索的时候，再把索引中 norm 值decode成一个float值，在这个过程中会有精度丢失，不保证可逆，如decode(encode(0.89)) = 0.75。

#### 词频tf(t in d) 计算公式(默认DefaultSimilarity )

$$tf(t\ in\ d )=sqrt{frequency}$$

#### 逆词频idf(t)计算公式(默认DefaultSimilarity )

$$idf=1+\log\frac{numDocs}{docFreq+1}$$

#### coord(q,d) 计算公式(DefaultSimilarity)

$$coord(q,d)=\frac{overlap}{maxOverlap}$$

#### queryNorm(q)计算公式 (DefaultSimilarity)

$$ queryNorm(q)=queryNorm(sumOfSquaredWeights)=\frac{1}{\sqrt{sumOfSquaredWeights}}$$

#### 这里sumOfSquaredWeights的计算公式为:

$$sumOfSquaredWeights= q.getBoost() ^2*\sum_{t\ in\ q}{(idf(t)* t.getBoost())^2}$$

#### norm(t,d)的计算公式为:

$$norm(t,d)= lengthNorm*\prod_{field\ f\ in\ d\ named\ as\ t}f.boost() $$
