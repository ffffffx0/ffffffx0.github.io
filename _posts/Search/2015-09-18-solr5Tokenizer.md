---
layout: post
title: "Solr5自定义逗号分词器"
keywords: ["distributed","Search"]
description: "Solr5.x &&SolrCloud"
category: "distributed"
tags: ["distributed","Search"]
---
#### 参考WhitespaceTokenizer写一个叫CommaTokenizer的Tokenizer，继承CharTokenizer

```
package com.xxx.yyy.zzz.analyzer;


import org.apache.lucene.analysis.core.WhitespaceTokenizerFactory;
import org.apache.lucene.analysis.util.CharTokenizer;
import org.apache.lucene.util.AttributeFactory;

/**
 * CommaTokenizer
 * 
 * may see {@link WhitespaceTokenizerFactory}
 */
public final class CommaTokenizer extends CharTokenizer {

	public CommaTokenizer() {
	}

	public CommaTokenizer(AttributeFactory factory) {
		super(factory);
	}

	@Override
	protected boolean isTokenChar(int c) {
		return !(c == 44);
		// return !Character.isWhitespace(c);
	}

}
```
参考WhitespaceTokenizer写一个叫CommaTokenizerFactory的TokenizerFactory，继承TokenizerFactory

```
package com.xxx.yyy.xxx.analyzer;

import org.apache.lucene.analysis.core.WhitespaceTokenizerFactory;
import org.apache.lucene.analysis.util.TokenizerFactory;
import org.apache.lucene.util.AttributeFactory;

import java.util.Map;

public class CommaTokenizerFactory extends TokenizerFactory {

  public CommaTokenizerFactory(Map<String,String> args) {
    super(args);
    if (!args.isEmpty()) {
      throw new IllegalArgumentException("Unknown parameters: " + args);
    }
  }

  @Override
  public CommaTokenizer create(AttributeFactory factory) {
    return new CommaTokenizer(factory);
  }
}
```

schema定义与配置

```
<field name="comma_name" type="comma_str" indexed="true" stored="true" omitNorms="true"/>
<field name="comma_pattern_name" type="comma_pattern" indexed="true" stored="true" omitNorms="true"/>

<fieldType name="comma_str" class="solr.TextField" positionIncrementGap="100">
    <analyzer>
        <tokenizer class="com.xxx.yyy.zzz.CommaTokenizerFactory"/>
    </analyzer>
</fieldType>

<fieldType name="comma_pattern" class="solr.TextField" positionIncrementGap="100">
    <analyzer>
        <tokenizer class="solr.PatternTokenizerFactory" pattern=", *" />  
    </analyzer>
</fieldType>

```

没错，这玩意也可以用PatternTokenizerFactory搞定

>
[ElasticSearch+Solr几个案例笔记](http://blog.csdn.net/u010454030/article/details/52625868)
