---
title: Flink NoWindowJoin原理
tagline: ""
category : flink
layout: post
tags : [flink, streamsets, realtime]
---

示例：

```
object StreamTableExample {

  // *************************************************************************
  //     PROGRAM
  // *************************************************************************

  def main(args: Array[String]): Unit = {

    // set up execution environment
    val env = StreamExecutionEnvironment.getExecutionEnvironment
    val tEnv = StreamTableEnvironment.create(env)

    val orderA = env.fromCollection(Seq(
      OrderA(1L, "beer", 3),
      OrderA(1L, "diaper", 4),
      OrderA(3L, "rubber", 2)))
      //.toTable(tEnv,'aId,'product1,'amount1)

    val orderB = env.fromCollection(Seq(
      OrderB(1L, "pen", 3),
      OrderB(2L, "rubber", 3),
      OrderB(4L, "beer", 1)))
      //.toTable(tEnv,'bId,'product2,'amount2)

    tEnv.registerDataStream("orderA",orderA)
    tEnv.registerDataStream("orderB",orderB)
    val result = tEnv.sqlQuery("select orderA.aId as aId,orderB.bId as bId, product1, amount1 from orderA  Inner join orderB on orderA.aId = orderB.bId" )
//    tEnv.registerDataStream("retTable",result.toRetractStream[OrderRet]);
//    // union the two tables
//    val result: DataStream[OrderRet] = orderA.join(orderB).where('aId === 'bId)
//      .select( 'aId,'bId,'product1, 'amount1)
//     // .where('amount > 2)
//      .toAppendStream[OrderRet]
//
//    tEnv.sqlQuery("select aId,bId,product1,amount1 from retTable where amount1>3")
    println(tEnv.explain(result))
    result.toRetractStream[OrderRet].print()

    env.execute()
  }

  // *************************************************************************
  //     USER DATA TYPES
  // *************************************************************************

  case class OrderA(aId: Long, product1: String, amount1: Int)

  case class OrderB(bId: Long, product2: String, amount2: Int)

  case class OrderRet(aId: Long,bId: Long, product1: String, amount1: Int)

}
```

sql语句：

```
select orderA.aId as aId,orderB.bId as bId, product1, amount1 from orderA  Inner join orderB on orderA.aId = orderB.bId
```
执行计划：

```
== Abstract Syntax Tree ==
LogicalProject(aId=[$0], bId=[$3], product1=[$1], amount1=[$2])
  LogicalJoin(condition=[=($0, $3)], joinType=[inner])
    LogicalTableScan(table=[[default_catalog, default_database, orderA]])
    LogicalTableScan(table=[[default_catalog, default_database, orderB]])

== Optimized Logical Plan ==
DataStreamCalc(select=[aId, bId, product1, amount1])
  DataStreamJoin(where=[=(aId, bId)], join=[aId, product1, amount1, bId], joinType=[InnerJoin])
    DataStreamScan(id=[1], fields=[aId, product1, amount1])
    DataStreamCalc(select=[bId])
      DataStreamScan(id=[2], fields=[bId, product2, amount2])

== Physical Execution Plan ==
Stage 1 : Data Source
	content : collect elements with CollectionInputFormat

Stage 2 : Data Source
	content : collect elements with CollectionInputFormat

	Stage 3 : Operator
		content : from: (aId, product1, amount1)
		ship_strategy : REBALANCE

		Stage 4 : Operator
			content : from: (bId, product2, amount2)
			ship_strategy : REBALANCE

			Stage 5 : Operator
				content : select: (bId)
				ship_strategy : FORWARD

				Stage 8 : Operator
					content : InnerJoin(where: (=(aId, bId)), join: (aId, product1, amount1, bId))
					ship_strategy : HASH

					Stage 9 : Operator
						content : select: (aId, bId, product1, amount1)
						ship_strategy : FORWARD

```
这里debug可以看下Optimized Logical Plan   
![No_Window_Join](https://raw.githubusercontent.com/2pc/2pc.github.io/master/images/no_window_join.png)

看下DataStreamJoin的translateToPlan
![DataStreamJoin——translateToPlan](https://raw.githubusercontent.com/2pc/2pc.github.io/master/images/join_translateToPlan.png)   
其中比较重要的还是joinOperator，这里是LegacyKeyedCoProcessOperator,   顺便看下LegacyKeyedCoProcessOperator   
![DataStreamJoin——translateToPlan](https://raw.githubusercontent.com/2pc/2pc.github.io/master/images/LegacyKeyedCoProcessOperator.png)   
LegacyKeyedCoProcessOperator实现TwoInputStreamOperator接口，实现了
processElement1(StreamRecord<IN1> var1)和processElement2(StreamRecord<IN2> var1)，

```
public void processElement1(StreamRecord<IN1> element) throws Exception {
	this.collector.setTimestamp(element);
	this.context.element = element;
	((CoProcessFunction)this.userFunction).processElement1(element.getValue(), this.context, this.collector);
	this.context.element = null;
}

public void processElement2(StreamRecord<IN2> element) throws Exception {
	this.collector.setTimestamp(element);
	this.context.element = element;
	((CoProcessFunction)this.userFunction).processElement2(element.getValue(), this.context, this.collector);
	this.context.element = null;
}
```
这里的userFunction可以是NonWindowLeftRightJoin，NonWindowInnerJoin，NonWindowLeftRightJoinWithNonEquiPredicates，NonWindowFullJoin等，
具体可以看DataStreamJoinToCoProcessTranslator的createJoinOperator方法，依据不同的join类型生成不同的userFunction   
直接看NonWindowLeftRightJoin的processElement

```
override def processElement(
	value: CRow,
	ctx: CoProcessFunction[CRow, CRow, CRow]#Context,
	out: Collector[CRow],
	currentSideState: MapState[Row, JTuple2[Long, Long]],
	otherSideState: MapState[Row, JTuple2[Long, Long]],
	recordFromLeft: Boolean): Unit = {

	val inputRow = value.row
	updateCurrentSide(value, ctx, currentSideState)

	cRowWrapper.reset()
	cRowWrapper.setCollector(out)
	cRowWrapper.setChange(value.change)

	// join other side data
	if (recordFromLeft == isLeftJoin) {
	preservedJoin(inputRow, recordFromLeft, otherSideState)
	} else {
	retractJoin(value, recordFromLeft, currentSideState, otherSideState)
	}
}
```
preservedJoin之后会调用到callJoinFunction

```
protected def callJoinFunction(
		inputRow: Row,
		inputRowFromLeft: Boolean,
		otherSideRow: Row,
		cRowWrapper: Collector[Row]): Unit = {

		if (inputRowFromLeft) {
		joinFunction.join(inputRow, otherSideRow, cRowWrapper)
		} else {
		joinFunction.join(otherSideRow, inputRow, cRowWrapper)
		}
}
```

这里的joinFunction是通过gencode生成的，将之前LegacyKeyedCoProcessOperator里的genJoinFuncCode,copy出来大概是这样的
```

public class DataStreamJoinRule$25 extends org.apache.flink.api.common.functions.RichFlatJoinFunction {

  
  final org.apache.flink.types.Row out =
      new org.apache.flink.types.Row(4);
  
  

  public DataStreamJoinRule$25() throws Exception {
    
    
  }

  
  

  @Override
  public void open(org.apache.flink.configuration.Configuration parameters) throws Exception {
    
    
  }

  @Override
  public void join(Object _in1, Object _in2, org.apache.flink.util.Collector c) throws Exception {
    org.apache.flink.types.Row in1 = (org.apache.flink.types.Row) _in1;
    org.apache.flink.types.Row in2 = (org.apache.flink.types.Row) _in2;
    
    boolean isNull$22 = (java.lang.Integer) in1.getField(2) == null;
    int result$21;
    if (isNull$22) {
      result$21 = -1;
    }
    else {
      result$21 = (java.lang.Integer) in1.getField(2);
    }
    
    
    boolean isNull$18 = (java.lang.Long) in1.getField(0) == null;
    long result$17;
    if (isNull$18) {
      result$17 = -1L;
    }
    else {
      result$17 = (java.lang.Long) in1.getField(0);
    }
    
    
    boolean isNull$24 = (java.lang.Long) in2.getField(0) == null;
    long result$23;
    if (isNull$24) {
      result$23 = -1L;
    }
    else {
      result$23 = (java.lang.Long) in2.getField(0);
    }
    
    
    boolean isNull$20 = (java.lang.String) in1.getField(1) == null;
    java.lang.String result$19;
    if (isNull$20) {
      result$19 = "";
    }
    else {
      result$19 = (java.lang.String) (java.lang.String) in1.getField(1);
    }
    
    
    
    
    
    
    
    if (isNull$18) {
      out.setField(0, null);
    }
    else {
      out.setField(0, result$17);
    }
    
    
    
    if (isNull$20) {
      out.setField(1, null);
    }
    else {
      out.setField(1, result$19);
    }
    
    
    
    if (isNull$22) {
      out.setField(2, null);
    }
    else {
      out.setField(2, result$21);
    }
    
    
    
    if (isNull$24) {
      out.setField(3, null);
    }
    else {
      out.setField(3, result$23);
    }
    
    c.collect(out);
    
  }

  @Override
  public void close() throws Exception {
    
    
  }
}

```

这是一个RichFlatJoinFunction函数，主要是将两个流中的需要的字段提取生成一个Row?


在经过sql解析生成AST以及各种逻辑/物理计划优化后最后调用toDataStream/toRetractStream

这里会包装一串transformations，于StreamGraph生成有关

```
  private def toDataStream[T](
      table: Table,
      modifyOperation: OutputConversionModifyOperation)
    : DataStream[T] = {
    val transformations = planner
      .translate(Collections.singletonList(modifyOperation))
    val streamTransformation: Transformation[T] = getTransformation(
      table,
      transformations)
    scalaExecutionEnvironment.getWrappedStreamExecutionEnvironment.addOperator(streamTransformation)
    new DataStream[T](new JDataStream[T](
      scalaExecutionEnvironment
        .getWrappedStreamExecutionEnvironment, streamTransformation))
  }
```
这个addOperator添加到transformations
```
public void addOperator(Transformation<?> transformation) {
    Preconditions.checkNotNull(transformation, "transformation must not be null.");
    this.transformations.add(transformation);
}
```
transformations在后续会用来生成StreamGraph

```
   public StreamGraph getStreamGraph(String jobName) {
        return this.getStreamGraphGenerator().setJobName(jobName).generate();
    }

    private StreamGraphGenerator getStreamGraphGenerator() {
        if (this.transformations.size() <= 0) {
            throw new IllegalStateException("No operators defined in streaming topology. Cannot execute.");
        } else {
            return (new StreamGraphGenerator(this.transformations, this.config, this.checkpointCfg)).setStateBackend(this.defaultStateBackend).setChaining(this.isChainingEnabled).setUserArtifacts(this.cacheFile).setTimeCharacteristic(this.timeCharacteristic).setDefaultBufferTimeout(this.bufferTimeout);
        }
    }
```
这里展示了示例sql的transformations：   
![transformation1](https://raw.githubusercontent.com/2pc/2pc.github.io/master/images/transformation2.png)   
![transformation2](https://raw.githubusercontent.com/2pc/2pc.github.io/master/images/transformation1.png)
