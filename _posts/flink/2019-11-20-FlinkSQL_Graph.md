---
title: Flink Graph转换
tagline: ""
category : flink
layout: post
tags : [flink, streamsets, realtime]
---

transformations-->streamGraph(Operator)-->JobGraph-->ExecutionGraph

## StreamGraph(Operator)-->JobGraph

```
//StreamExecutionEnvironment
def execute() = javaEnv.execute()
public JobExecutionResult execute() throws Exception {
    return this.execute("Flink Streaming Job");
}

public JobExecutionResult execute(String jobName) throws Exception {
    Preconditions.checkNotNull(jobName, "Streaming Job name should not be null.");
    return this.execute(this.getStreamGraph(jobName));
}
//LocalStreamEnvironment
public JobExecutionResult execute(StreamGraph streamGraph) throws Exception {
  JobGraph jobGraph = streamGraph.getJobGraph();
  ...
}
```
LocalStreamEnvironment的话，直接调用streamGraph的getJobGraph()的到JobGraph
```
  public JobGraph getJobGraph() {
      return this.getJobGraph((JobID)null);
  }
public JobGraph getJobGraph(@Nullable JobID jobID) {
  if (this.isIterative() && this.checkpointConfig.isCheckpointingEnabled() && !this.checkpointConfig.isForceCheckpointing()) {
      throw new UnsupportedOperationException("Checkpointing is currently not supported by default for iterative jobs, as we cannot guarantee exactly once semantics. State checkpoints happen normally, but records in-transit during the snapshot will be lost upon failure. \nThe user can force enable state checkpoints with the reduced guarantees by calling: env.enableCheckpointing(interval,true)");
  } else {
      return StreamingJobGraphGenerator.createJobGraph(this, jobID);
  }
}
```
主要下StreamingJobGraphGenerator生成
```
StreamingJobGraphGenerator.createJobGraph(this, jobID);
```
再看下remote的吧
```
//RemoteStreamEnvironment
public JobExecutionResult execute(StreamGraph streamGraph) throws Exception {
    this.transformations.clear();
    return this.executeRemotely(streamGraph, this.jarFiles);
}
//RemoteStreamEnvironment
@Deprecated
protected JobExecutionResult executeRemotely(StreamGraph streamGraph, List<URL> jarFiles) throws ProgramInvocationException {
    return executeRemotely(streamGraph, this.getClass().getClassLoader(), this.getConfig(), jarFiles, this.host, this.port, this.clientConfiguration, this.globalClasspaths, this.savepointRestoreSettings);
}
//RemoteStreamEnvironment
private static JobExecutionResult executeRemotely(StreamGraph streamGraph, ClassLoader envClassLoader, ExecutionConfig executionConfig, List<URL> jarFiles, String host, int port, Configuration clientConfiguration, List<URL> globalClasspaths, SavepointRestoreSettings savepointRestoreSettings) throws ProgramInvocationException {
...
var12 = client.run(streamGraph, jarFiles, globalClasspaths, userCodeClassLoader, savepointRestoreSettings).getJobExecutionResult();
...
}
//ClusterClient
public JobSubmissionResult run(FlinkPlan compiledPlan, List<URL> libraries, List<URL> classpaths, ClassLoader classLoader, SavepointRestoreSettings savepointSettings) throws ProgramInvocationException {
    JobGraph job = getJobGraph(this.flinkConfig, compiledPlan, libraries, classpaths, savepointSettings);
    return this.submitJob(job, classLoader);
}
//ClusterClient
public static JobGraph getJobGraph(Configuration flinkConfig, FlinkPlan optPlan, List<URL> jarFiles, List<URL> classpaths, SavepointRestoreSettings savepointSettings) {
    JobGraph job;
    if (optPlan instanceof StreamingPlan) {
        job = ((StreamingPlan)optPlan).getJobGraph();
        job.setSavepointRestoreSettings(savepointSettings);
    } else {
        JobGraphGenerator gen = new JobGraphGenerator(flinkConfig);
        job = gen.compileJobGraph((OptimizedPlan)optPlan);
    }
    ...
}

```
注意下 StreamGraph extends StreamingPlan，所以这里就是StreamGraph

## JobGraph-->ExecutionGraph LegacyScheduler
```
//JobMaster
schedulerAssignedFuture.thenRun(this::startScheduling);
//JobMaster
private void startScheduling() {
  checkState(jobStatusListener == null);
  // register self as job status change listener
  jobStatusListener = new JobManagerJobStatusListener();
  schedulerNG.registerJobStatusListener(jobStatusListener);

  schedulerNG.startScheduling();
}
//LegacyScheduler
public void startScheduling() {
  mainThreadExecutor.assertRunningInMainThread();

  try {
    executionGraph.scheduleForExecution();
  }
  catch (Throwable t) {
    executionGraph.failGlobal(t);
  }
}

```
看下executionGraph的构造
```
//LegacyScheduler构造函数
this.executionGraph = createAndRestoreExecutionGraph(jobManagerJobMetricGroup, checkNotNull(shuffleMaster), checkNotNull(partitionTracker));
//LegacyScheduler.createAndRestoreExecutionGraph
private ExecutionGraph createAndRestoreExecutionGraph(
    JobManagerJobMetricGroup currentJobManagerJobMetricGroup,
    ShuffleMaster<?> shuffleMaster,
    PartitionTracker partitionTracker) throws Exception {

  ExecutionGraph newExecutionGraph = createExecutionGraph(currentJobManagerJobMetricGroup, shuffleMaster, partitionTracker);

  final CheckpointCoordinator checkpointCoordinator = newExecutionGraph.getCheckpointCoordinator();

  if (checkpointCoordinator != null) {
    // check whether we find a valid checkpoint
    if (!checkpointCoordinator.restoreLatestCheckpointedState(
      newExecutionGraph.getAllVertices(),
      false,
      false)) {

      // check whether we can restore from a savepoint
      tryRestoreExecutionGraphFromSavepoint(newExecutionGraph, jobGraph.getSavepointRestoreSettings());
    }
  }

  return newExecutionGraph;
}
```

link:
[Flink Job提交流程(Dispatcher之后)](https://blog.csdn.net/lisenyeahyeah/article/details/100662367)
