---
title: Spark SQL Analyzer
tagline: "spark"
category : spark
layout: post
tags : [Spark Sql]
---

Analyzer结合 catalog 进行绑定,生成 Resolved Logical Plan，在其父类RuleExecutor中匹配相应规则

```
lazy val analyzed: LogicalPlan = {
  SparkSession.setActiveSession(sparkSession)
  sparkSession.sessionState.analyzer.executeAndCheck(logical)
}
def executeAndCheck(plan: LogicalPlan): LogicalPlan = AnalysisHelper.markInAnalyzer {
  val analyzed = execute(plan)
  try {
    checkAnalysis(analyzed)
    analyzed
  } catch {
    case e: AnalysisException =>
      val ae = new AnalysisException(e.message, e.line, e.startPosition, Option(analyzed))
      ae.setStackTrace(e.getStackTrace)
      throw ae
  }
}
override def execute(plan: LogicalPlan): LogicalPlan = {
AnalysisContext.reset()
try {
  executeSameContext(plan)
} finally {
  AnalysisContext.reset()
}
}
private def executeSameContext(plan: LogicalPlan): LogicalPlan = super.execute(plan)
```
RuleExecutor.execute()

```
def execute(plan: TreeType): TreeType = {
  var curPlan = plan
  val queryExecutionMetrics = RuleExecutor.queryExecutionMeter

  batches.foreach { batch =>
```
