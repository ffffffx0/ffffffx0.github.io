---
layout: post
title: "Gradient Boosting Decision Tree 相关笔记整理"
keywords: ["ML","Datascience","LambdaMART","RankNet","RankBoost","GBDT"]
description: "Datascience Guide"
category: "Datascience"
tags: ["ML","Datascience"]
---
#### 首先论文

[Greedy Function Approximation: A Gradient Boosting Machine.](https://statweb.stanford.edu/~jhf/ftp/trebst.pdf)

####  一些相关的开源实现

先贴几套开源实现代码的地址,这里主要研究的2,3，其中2是c++版的残差版本,3中的MART也是残差版本实现，最近在做ReRank相关的事情刚好要用到LambdaMART

>
1. Xgboost[Xgboost源码-github](https://github.com/dmlc/xgboost/tree/master/)  [Xgboost文档](https://xgboost.readthedocs.io/en/latest/)
2. C++版 gbdt[源码下载地址-CSDN 10积分](http://download.csdn.net/detail/w28971023/4837775)             [github修改版地址](https://github.com/2pc/libgbdt)
3. Ranklib[sourceforge地址](https://sourceforge.net/p/lemur/wiki/RankLib/)
4. simple-gbdt[google code地址](https://code.google.com/archive/p/simple-gbdt/) 依赖tbb库[github 某同学fork版本](https://github.com/hcy0807/simple-gbdt)
5. elf项目[sourceforge地址](http://elf-project.sourceforge.net/)
6. Spark中的GradientBoostedTrees[github地址](https://github.com/apache/spark/blob/master/mllib/src/main/scala/org/apache/spark/mllib/tree/GradientBoostedTrees.scala)


暂且简单描述下

>
* 1中Xgboost支持力度很大，支持python,R，Java.etc 甚至spark
* 2中所指gbdt网上有几篇分析的文章都是用的这个版本，这个版本训练是没啥问题，不过predict的时候不友好，感觉简化了。修改了下
* 3中Ranklib支持的算法也很多，基本可以开包即用了.
* 4中simple-gbdt，依赖tbb库.
* Spark中的实现

讲到GBDT的时候首先应该指出是残差版本还是Gradient版本，因为在原理，求解，实现上存在一些差异（这个差异在理解上可能会导致犯迷糊,反正自己绕不少弯路）,这里主要讨论残差版本。xgboost目前也在使用，还没深入研究，这里主要研究2和3中的版本。2，3也有讲解源代码的文章了。Ranklib的实现比较好理解。

#### Future选取问题

可以随机选择rand_fea_num个特征进行分裂，确定最优的分裂特性
2中C++代码实现

```
for (int i = 0; i < gbdt_inf.rand_fea_num; ++i) 
 {
  int select = rand() % (last+1);//随机选择一个特征
 }
```

#### 分裂问题，

分裂：分裂后的均方误差最小

Best split标准计算依据

计算公式：

![游园惊梦-GBDT原理实例演示 2](http://images.cnitblog.com/blog/61573/201503/251821497246538.png)

2中C++代码实现

```
for (int j=ninf.index_b; j< ninf.index_e; j++)
{
// d = y_result_score[data_set->order_i[j]];
d = data_set->y_list[data_set->order_i[j]];
  left_sum += d;
  right_sum -= d;
  left_num++;
  right_num--;
if (data_set->fv[j] < data_set->fv[j+1])
  {/** 均方误差Mean Squared Error, MSE）最小? */
	crit = (left_sum * left_sum / left_num) + (right_sum * right_sum / right_num) - ninf.critparent;
  	if (crit > critvar) 
	{
		tmpsplit = (data_set->fv[j] + data_set->fv[j+1]) / 2.0; // 实际分割用的feature value
		critvar = crit;
	}
  }
}

if (critvar > critmax) // 如果这个feature最终的critvar > cirtmax, 保存信息
{
spinf->bestsplit = tmpsplit; // split feature vale
  spinf->bestid = fid; // split feature id
  critmax = critvar; // split crit vaule
}
```


#### 节点的输出值

节点的输出值为该节点上所有sample的label的均值
Ranklib中MART的实现,在GBDT中pseudoResponses就是残差，也就是label

```
protected void updateTreeOutput(RegressionTree rt)
{
	List<Split> leaves = rt.leaves();
	for(int i=0;i<leaves.size();i++)
	{
		float s1 = 0.0F;
		Split s = leaves.get(i);
		int[] idx = s.getSamples();
		
		System.out.println("leaves(i)" + i +" idx.length: "+idx.length);
		for(int j=0;j<idx.length;j++)
		{
			int k = idx[j];
			s1 += pseudoResponses[k];
		}
		s.setOutput(s1/idx.length);
	}
}
```

2中的代码gbdt_test.cpp 中跑test的时候尽然是一行输入一行输出？  修改了几行代码

```
char* test_file_name = argv[2];
std::ifstream fin(test_file_name,std::ios::in)
while(getline(fin,line))
```
进入gbdt的目录

```
cd lib_gbdt
make all
cd output/test
./gbdt-train -r 0.8 -t 100 -s 0.03 -n 30 -d 5 -m test.model -f ../../train
./gbdt-test ./test.model ../../train
```

GDB调试代码

```
gdb  ./gbdt-train  
```
可以设置参数,断点，执行

```
(gdb) set args -r 0.8 -t 100 -s 0.03 -n 30 -d 5 -m test.model -f ../../train 
(gdb) show args
```

参考链接

>
* [GBDT代码解读](http://blog.sina.com.cn/s/blog_4d1865f00101bbtl.html)
* [GBDT源码剖析](http://blog.csdn.net/w28971023/article/details/8249108)
* [GBDT算法整理](http://blog.csdn.net/davidie/article/details/50897278)
* [Learning To Rank之LambdaMART的前世今生](http://blog.csdn.net/huagong_adu/article/details/40710305?utm_source=tuicool&utm_medium=referral)
* [RankLib源码分析](http://blog.csdn.net/guoguo881218/article/category/2805459)
* [理解GBDT算法（一）——理论](http://blog.csdn.net/puqutogether/article/details/41957089)
* [理解GBDT算法（二）——基于残差的版本](http://blog.csdn.net/puqutogether/article/details/44752611)
* [理解GBDT算法（三）——基于梯度的版本](http://blog.csdn.net/puqutogether/article/details/44781035)   
* [求解分裂问题参考李航博士统计学习方法-Machine Learning & Algorithm 决策树与迭代决策树（GBDT）](http://www.cnblogs.com/maybe2030/p/4734645.html#_label4)   
