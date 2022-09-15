---
title: StrucuredStreaming window
tagline: "spark"
category : spark
layout: post
tags : [Spark]
---

打印窗口示例
```
import org.apache.spark.sql.streaming.{OutputMode, ProcessingTime}
import org.apache.spark.sql.{DataFrame, SparkSession}
import org.apache.spark.sql.functions._

object StructuredNetworkWindowShow {

  def main(args: Array[String]) {
    val host = "10.9.22.136"
    val port = "6666"
    val windowSize = 3
    val slideSize = 2
    val triggerTime = 1
    if (slideSize > windowSize) {
      System.err.println("<slide duration> must be less than or equal to <window duration>")
    }
    val windowDuration = s"$windowSize seconds"
    val slideDuration = s"$slideSize seconds"

    val spark = SparkSession
      .builder
      .master("local")
      .appName("StructuredNetworkWindowShow"+String.valueOf(System.currentTimeMillis()))
      .getOrCreate()

    import spark.implicits._
    // Create DataFrame representing the stream of input lines from connection to host:port
    val lines = spark.readStream
      .format("socket")
      .option("host", host)
      .option("port", port)
      .option("includeTimestamp", true)
      .load()
    val wordCounts:DataFrame = lines.select(window($"timestamp",windowDuration,slideDuration),$"value")
    // Start running the query that prints the windowed word counts to the console

    val query = wordCounts.writeStream
      .outputMode(OutputMode.Append())
      .format("console")
      .trigger(ProcessingTime(s"$triggerTime seconds"))
      .option("truncate", "false")
      .start()

    query.awaitTermination()

  }
```

如下window大小为3，滑动步长2，triggerTime=1
```
-------------------------------------------
Batch: 3
-------------------------------------------
+------------------------------------------+-----+
|window                                    |value|
+------------------------------------------+-----+
|[2019-08-27 14:20:36, 2019-08-27 14:20:39]|11   |
|[2019-08-27 14:20:38, 2019-08-27 14:20:41]|11   |
|[2019-08-27 14:20:38, 2019-08-27 14:20:41]|12   |
|[2019-08-27 14:20:40, 2019-08-27 14:20:43]|12   |
|[2019-08-27 14:20:40, 2019-08-27 14:20:43]|13   |
+------------------------------------------+-----+
-------------------------------------------
Batch: 6
-------------------------------------------
19/08/27 14:26:39 INFO WriteToDataSourceV2Exec: Data source writer org.apache.spark.sql.execution.streaming.sources.MicroBatchWriter@37e8844 committed.
+------------------------------------------+-----+
|window                                    |value|
+------------------------------------------+-----+
|[2019-08-27 14:26:32, 2019-08-27 14:26:35]|17   |
|[2019-08-27 14:26:34, 2019-08-27 14:26:37]|18   |
|[2019-08-27 14:26:34, 2019-08-27 14:26:37]|19   |
|[2019-08-27 14:26:36, 2019-08-27 14:26:39]|19   |
+------------------------------------------+-----+
```
