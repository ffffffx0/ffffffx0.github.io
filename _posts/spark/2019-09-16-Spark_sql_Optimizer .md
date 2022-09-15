---
title: Spark SQL Optimizer
tagline: "spark"
category : spark
layout: post
tags : [Spark Sql]
---
Optimizer与一样，也是继承自RuleExecutor,匹配规则对SQL进行优化

```
lazy val withCachedData: LogicalPlan = {
  assertAnalyzed()
  assertSupported()
  sparkSession.sharedState.cacheManager.useCachedData(analyzed)
}

lazy val optimizedPlan: LogicalPlan = sparkSession.sessionState.optimizer.execute(withCachedData)
```

```
def execute(plan: TreeType): TreeType = {
  var curPlan = plan
  val queryExecutionMetrics = RuleExecutor.queryExecutionMeter

  batches.foreach { batch =>
    val batchStartPlan = curPlan
    var iteration = 1
    var lastPlan = curPlan
    var continue = true

    // Run until fix point (or the max number of iterations as specified in the strategy.
    while (continue) {
      curPlan = batch.rules.foldLeft(curPlan) {
        case (plan, rule) =>
          val startTime = System.nanoTime()
          val result = rule(plan)
          val runTime = System.nanoTime() - startTime

          if (!result.fastEquals(plan)) {
            queryExecutionMetrics.incNumEffectiveExecution(rule.ruleName)
            queryExecutionMetrics.incTimeEffectiveExecutionBy(rule.ruleName, runTime)
            logTrace(
              s"""
                |=== Applying Rule ${rule.ruleName} ===
                |${sideBySide(plan.treeString, result.treeString).mkString("\n")}
              """.stripMargin)
          }
          queryExecutionMetrics.incExecutionTimeBy(rule.ruleName, runTime)
          queryExecutionMetrics.incNumExecution(rule.ruleName)

          // Run the structural integrity checker against the plan after each rule.
          if (!isPlanIntegral(result)) {
            val message = s"After applying rule ${rule.ruleName} in batch ${batch.name}, " +
              "the structural integrity of the plan is broken."
            throw new TreeNodeException(result, message, null)
          }

          result
      }
      iteration += 1
      if (iteration > batch.strategy.maxIterations) {
        // Only log if this is a rule that is supposed to run more than once.
        if (iteration != 2) {
          val message = s"Max iterations (${iteration - 1}) reached for batch ${batch.name}"
          if (Utils.isTesting) {
            throw new TreeNodeException(curPlan, message, null)
          } else {
            logWarning(message)
          }
        }
        continue = false
      }

      if (curPlan.fastEquals(lastPlan)) {
        logTrace(
          s"Fixed point reached for batch ${batch.name} after ${iteration - 1} iterations.")
        continue = false
      }
      lastPlan = curPlan
    }

    if (!batchStartPlan.fastEquals(curPlan)) {
      logDebug(
        s"""
          |=== Result of Batch ${batch.name} ===
          |${sideBySide(batchStartPlan.treeString, curPlan.treeString).mkString("\n")}
        """.stripMargin)
    } else {
      logTrace(s"Batch ${batch.name} has no effect.")
    }
  }

  curPlan
}
```
