---
title: Spark SQL Planner
tagline: "spark"
category : spark
layout: post
tags : [Spark Sql]
---
Optimizer优化后最终得到的还是Logical Plan，Planner会将Logical Plan转化为 Physical Plan

```
lazy val sparkPlan: SparkPlan = {
  SparkSession.setActiveSession(sparkSession)
  // TODO: We use next(), i.e. take the first plan returned by the planner, here for now,
  //       but we will implement to choose the best plan.
  planner.plan(ReturnAnswer(optimizedPlan)).next()
}
```
QueryPlanner.plan
```
def plan(plan: LogicalPlan): Iterator[PhysicalPlan] = {
  // Obviously a lot to do here still...

  // Collect physical plan candidates.
  val candidates = strategies.iterator.flatMap(_(plan))

  // The candidates may contain placeholders marked as [[planLater]],
  // so try to replace them by their child plans.
  val plans = candidates.flatMap { candidate =>
    val placeholders = collectPlaceholders(candidate)

    if (placeholders.isEmpty) {
      // Take the candidate as is because it does not contain placeholders.
      Iterator(candidate)
    } else {
      // Plan the logical plan marked as [[planLater]] and replace the placeholders.
      placeholders.iterator.foldLeft(Iterator(candidate)) {
        case (candidatesWithPlaceholders, (placeholder, logicalPlan)) =>
          // Plan the logical plan for the placeholder.
          val childPlans = this.plan(logicalPlan)

          candidatesWithPlaceholders.flatMap { candidateWithPlaceholders =>
            childPlans.map { childPlan =>
              // Replace the placeholder by the child plan
              candidateWithPlaceholders.transformUp {
                case p if p.eq(placeholder) => childPlan
              }
            }
          }
      }
    }
  }

  val pruned = prunePlans(plans)
  assert(pruned.hasNext, s"No plan for $plan")
  pruned
}
```
这里strategies

```
override def strategies: Seq[Strategy] =
  experimentalMethods.extraStrategies ++
    extraPlanningStrategies ++ (
    PythonEvals ::
    DataSourceV2Strategy ::
    FileSourceStrategy ::
    DataSourceStrategy(conf) ::
    SpecialLimits ::
    Aggregation ::
    Window ::
    JoinSelection ::
    InMemoryScans ::
    BasicOperators :: Nil)
```
