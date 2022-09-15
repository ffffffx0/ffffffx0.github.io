---
title: Spark SQL QueryExecution
tagline: "spark"
category : spark
layout: post
tags : [Spark Sql]
---

1.sql 通过SqlParser 解析成 Unresolved Logical Plan;   
2.analyzer 结合catalog 进行绑定,生成 Logical Plan;   
3.optimizer 对 Logical Plan 优化,生成 Optimized LogicalPlan;   
4.SparkPlan 将 Optimized LogicalPlan 转换成 Physical Plan;   
5.prepareForExecution()将 Physical Plan 转换成 executed Physical Plan;   
6.execute()执行可执行物理计划，得到RDD;

```
  override def addBatch(batchId: Long, data: DataFrame): Unit = {
    val resolvedEncoder = encoder.resolveAndBind(
      data.logicalPlan.output,
      data.sparkSession.sessionState.analyzer)
    val rdd = data.queryExecution.toRdd.map[T](resolvedEncoder.fromRow)(encoder.clsTag)
    val ds = data.sparkSession.createDataset(rdd)(encoder)
    batchWriter(ds, batchId)
  }
```
从toRdd往前推
```
  /** Internal version of the RDD. Avoids copies and has no schema */
  lazy val toRdd: RDD[InternalRow] = executedPlan.execute()
```
 executedPlan: SparkPlan
```
  // executedPlan should not be used to initialize any SparkPlan. It should be
  // only used for execution.
  lazy val executedPlan: SparkPlan = prepareForExecution(sparkPlan)
```
sparkPlan: SparkPlan
```
  lazy val sparkPlan: SparkPlan = {
    SparkSession.setActiveSession(sparkSession)
    // TODO: We use next(), i.e. take the first plan returned by the planner, here for now,
    //       but we will implement to choose the best plan.
    planner.plan(ReturnAnswer(optimizedPlan)).next()
  }
```
optimizedPlan: LogicalPlan
```
  lazy val optimizedPlan: LogicalPlan = sparkSession.sessionState.optimizer.execute(withCachedData)
```
withCachedData: LogicalPlan
```
  lazy val withCachedData: LogicalPlan = {
    assertAnalyzed()
    assertSupported()
    sparkSession.sharedState.cacheManager.useCachedData(analyzed)
  }

```
analyzed: LogicalPlan
```
  lazy val analyzed: LogicalPlan = {
    SparkSession.setActiveSession(sparkSession)
    sparkSession.sessionState.analyzer.executeAndCheck(logical)
  }

```
后续：

```
sc.runjob()
```
eg:
collect()
```
def executeCollect(): Array[InternalRow] = {
  val byteArrayRdd = getByteArrayRdd()

  val results = ArrayBuffer[InternalRow]()
  byteArrayRdd.collect().foreach { countAndBytes =>
    decodeUnsafeRows(countAndBytes._2).foreach(results.+=)
  }
  results.toArray
}
//RDD.scala
  def collect(): Array[T] = withScope {
  val results = sc.runJob(this, (iter: Iterator[T]) => iter.toArray)
  Array.concat(results: _*)
}
```
take
```
def executeTake(n: Int): Array[InternalRow] = {
if (n == 0) {
  return new Array[InternalRow](0)
}

val childRDD = getByteArrayRdd(n).map(_._2)

val buf = new ArrayBuffer[InternalRow]
val totalParts = childRDD.partitions.length
var partsScanned = 0
while (buf.size < n && partsScanned < totalParts) {
  // The number of partitions to try in this iteration. It is ok for this number to be
  // greater than totalParts because we actually cap it at totalParts in runJob.
  var numPartsToTry = 1L
  if (partsScanned > 0) {
    // If we didn't find any rows after the previous iteration, quadruple and retry.
    // Otherwise, interpolate the number of partitions we need to try, but overestimate
    // it by 50%. We also cap the estimation in the end.
    val limitScaleUpFactor = Math.max(sqlContext.conf.limitScaleUpFactor, 2)
    if (buf.isEmpty) {
      numPartsToTry = partsScanned * limitScaleUpFactor
    } else {
      val left = n - buf.size
      // As left > 0, numPartsToTry is always >= 1
      numPartsToTry = Math.ceil(1.5 * left * partsScanned / buf.size).toInt
      numPartsToTry = Math.min(numPartsToTry, partsScanned * limitScaleUpFactor)
    }
  }

  val p = partsScanned.until(math.min(partsScanned + numPartsToTry, totalParts).toInt)
  val sc = sqlContext.sparkContext
  val res = sc.runJob(childRDD,
    (it: Iterator[Array[Byte]]) => if (it.hasNext) it.next() else Array.empty[Byte], p)

  buf ++= res.flatMap(decodeUnsafeRows)

  partsScanned += p.size
}

if (buf.size > n) {
  buf.take(n).toArray
} else {
  buf.toArray
}
  }
```
