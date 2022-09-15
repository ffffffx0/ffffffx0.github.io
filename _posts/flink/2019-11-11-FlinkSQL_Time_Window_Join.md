---
title: Flink Time Window Join原理
tagline: ""
category : flink
layout: post
tags : [flink, streamsets, realtime]
---


rules:

blink:

```
FlinkStreamRuleSets
```
flink:

```
FlinkRuleSets
```
blink: StreamExecWindowJoin,StreamExecJoin


RowTimeBoundedStreamJoin   
继承自TimeBoundedStreamJoin，这个TimeBoundedStreamJoin(在早期名称TimeBoundedStreamInnerJoin,仅限innerjoin?)
ProcTimeBoundedStreamJoin
```
/**
 * A CoProcessFunction to execute time-bounded stream inner-join.
 * Two kinds of time criteria:
 * "L.time between R.time + X and R.time + Y" or "R.time between L.time - Y and L.time - X"
 * X and Y might be negative or positive and X <= Y.
 */
abstract class TimeBoundedStreamJoin extends CoProcessFunction<BaseRow, BaseRow, BaseRow> {}
 ```
IntervalJoinOperator   
StreamTwoInputProcessor

join 示例

```
package com.fenqile.flink.pvuv

import java.text.SimpleDateFormat
import java.util.Date

import org.apache.flink.streaming.api.TimeCharacteristic
import org.apache.flink.streaming.api.scala.StreamExecutionEnvironment
import org.apache.flink.table.api.scala.StreamTableEnvironment
import org.apache.flink.types.Row
import org.apache.flink.api.scala._
import org.apache.flink.streaming.api.functions.AssignerWithPunctuatedWatermarks
import org.apache.flink.streaming.api.watermark.Watermark
import org.apache.flink.table.api.scala._

import scala.collection.mutable

object TumbleTest {

  def main(args: Array[String]): Unit = {
    val env = StreamExecutionEnvironment.getExecutionEnvironment
    val tEnv = StreamTableEnvironment.create(env)
   // env.setStateBackend(getStateBackend)
    env.setStreamTimeCharacteristic(TimeCharacteristic.EventTime)
   // StreamITCase.clear
    val data1 = new mutable.MutableList[(String, String, Long)]
    data1.+=(("A", "L-7", 1570365660001L))
    data1.+=(("B", "L-7", 1570365665000L))
    data1.+=(("C", "L-7", 1572336777999L)) // no joining record

    data1.+=(("D", "L-7", 1572337111001L))
//    data1.+=(("D", "L-7", 1510365661000L))
//    data1.+=(("D", "L-7", 1510365665000L))

    val data2 = new mutable.MutableList[(String, String, Long)]
//    data2.+=(("A", "R-1", 7000L)) // 3 joining records
//    data2.+=(("B", "R-4", 7000L)) // 1 joining records
//    data2.+=(("A", "R-3", 8000L)) // 3 joining records
//    data2.+=(("D", "R-2", 8000L)) // no joining record
    data2.+=(("A", "R-2", 1570365660000L)) // no joining record
    data2.+=(("A", "R-2", 1570365663000L)) // no joining record
    data2.+=(("A", "R-2", 1570365664000L)) // no joining record
    data2.+=(("B", "R-2", 1570365665000L)) // no joining record1572336778
    data2.+=(("C", "R-2", 1572336778000L)) // no joining record
    data2.+=(("D", "R-2", 1572337111000L))
    val format = new SimpleDateFormat("yyyy-MM-dd HH:mm:ss.SSS")

    println("data1:")
    data1.map(x =>{
      val  date:Date =  new Date(x._3)
      println((x._1,x._2,format.format(date)))
    })
    println("data2:")
    data2.map(x =>{
      val  date:Date =  new Date(x._3)
      println((x._1,x._2,format.format(date)))
    })
    val t1 = env.fromCollection(data1)
      .assignTimestampsAndWatermarks(new Row3WatermarkExtractor2)
      .toTable(tEnv, 'key, 'id, 'rt.rowtime)
    val t2 = env.fromCollection(data2)
      .assignTimestampsAndWatermarks(new Row3WatermarkExtractor2)
      .toTable(tEnv, 'key, 'id, 'rt.rowtime)

    tEnv.registerTable("T1", t1)
    tEnv.registerTable("T2", t2)
    val sqlQuery =
      """
        |SELECT t2.key,TUMBLE_START(t2.rt, INTERVAL '4' SECOND),TUMBLE_END(t2.rt, INTERVAL '4' SECOND), COUNT(t1.key)
        |FROM T1 AS t1 join T2 AS t2 ON
        |  t1.key = t2.key AND
        |  t1.rt BETWEEN t2.rt - INTERVAL '3' SECOND AND
        |    t2.rt + INTERVAL '3' SECOND
        |GROUP BY TUMBLE(t2.rt, INTERVAL '4' SECOND),t2.key
        |""".stripMargin

    val result = tEnv.sqlQuery(sqlQuery)

    println(tEnv.explain(result))
    val ret=result.toAppendStream[Row]
    //System.out.println(tEnv.explain(result))
    ret.print()
    env.execute("")
  }

}
private class Row3WatermarkExtractor2
  extends AssignerWithPunctuatedWatermarks[(String, String, Long)] {

  override def checkAndGetNextWatermark(
                                         lastElement: (String, String, Long),
                                         extractedTimestamp: Long): Watermark = {
    new Watermark(extractedTimestamp - 1)
  }

  override def extractTimestamp(
                                 element: (String, String, Long),
                                 previousElementTimestamp: Long): Long = {
    element._3
  }

```
sql  explain:

```
== Abstract Syntax Tree ==
LogicalProject(key=[$1], EXPR$1=[TUMBLE_START($0)], EXPR$2=[TUMBLE_END($0)], EXPR$3=[$2])
  LogicalAggregate(group=[{0, 1}], EXPR$3=[COUNT($2)])
    LogicalProject($f0=[TUMBLE($5, 4000:INTERVAL SECOND)], key=[$3], $f2=[$0])
      LogicalJoin(condition=[AND(=($0, $3), >=($2, -($5, 3000:INTERVAL SECOND)), <=($2, +($5, 3000:INTERVAL SECOND)))], joinType=[inner])
        LogicalTableScan(table=[[default_catalog, default_database, T1]])
        LogicalTableScan(table=[[default_catalog, default_database, T2]])

== Optimized Logical Plan ==
DataStreamCalc(select=[key0 AS key, w$start AS EXPR$1, w$end AS EXPR$2, EXPR$3])
  DataStreamGroupWindowAggregate(groupBy=[key0], window=[TumblingGroupWindow('w$, 'rt0, 4000.millis)], select=[key0, COUNT(key) AS EXPR$3, start('w$) AS w$start, end('w$) AS w$end, rowtime('w$) AS w$rowtime, proctime('w$) AS w$proctime])
    DataStreamCalc(select=[rt0, key0, key])
      DataStreamWindowJoin(where=[AND(=(key, key0), >=(CAST(rt), -(CAST(rt0), 3000:INTERVAL SECOND)), <=(CAST(rt), +(CAST(rt0), 3000:INTERVAL SECOND)))], join=[key, rt, key0, rt0], joinType=[InnerJoin])
        DataStreamCalc(select=[key, rt])
          DataStreamScan(id=[2], fields=[key, id, rt])
        DataStreamCalc(select=[key, rt])
          DataStreamScan(id=[4], fields=[key, id, rt])

== Physical Execution Plan ==
Stage 1 : Data Source
	content : collect elements with CollectionInputFormat

	Stage 2 : Operator
		content : Timestamps/Watermarks
		ship_strategy : FORWARD

Stage 3 : Data Source
	content : collect elements with CollectionInputFormat

	Stage 4 : Operator
		content : Timestamps/Watermarks
		ship_strategy : FORWARD

		Stage 5 : Operator
			content : from: (key, id, rt)
			ship_strategy : REBALANCE

			Stage 6 : Operator
				content : select: (key, rt)
				ship_strategy : FORWARD

				Stage 7 : Operator
					content : from: (key, id, rt)
					ship_strategy : REBALANCE

					Stage 8 : Operator
						content : select: (key, rt)
						ship_strategy : FORWARD

						Stage 11 : Operator
							content : where: (AND(=(key, key0), >=(CAST(rt), -(CAST(rt0), 3000:INTERVAL SECOND)), <=(CAST(rt), +(CAST(rt0), 3000:INTERVAL SECOND)))), join: (key, rt, key0, rt0)
							ship_strategy : HASH

							Stage 12 : Operator
								content : select: (rt0, key0, key)
								ship_strategy : FORWARD

								Stage 13 : Operator
									content : time attribute: (rt0)
									ship_strategy : FORWARD

									Stage 15 : Operator
										content : groupBy: (key0), window: (TumblingGroupWindow('w$, 'rt0, 4000.millis)), select: (key0, COUNT(key) AS EXPR$3, start('w$) AS w$start, end('w$) AS w$end, rowtime('w$) AS w$rowtime, proctime('w$) AS w$proctime)
										ship_strategy : HASH

										Stage 16 : Operator
											content : select: (key0 AS key, w$start AS EXPR$1, w$end AS EXPR$2, EXPR$3)
											ship_strategy : FORWARD


```
