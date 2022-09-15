---
title: Flink CheckPoint实现
tagline: ""
category : flink
layout: post
tags : [flink, streamsets, realtime]
---

![Flink CheckPoint](https://gitee.com/beyondbat/notes/raw/master/image/ckp2.png)

生成Graph的时候，依据checkpoint的配置，会注册一个JobStatusListener，
```
ExecutionGraphBuilder.buildGraph-->executionGraph.enableCheckpointing
//
// interval of max long value indicates disable periodic checkpoint,
// the CheckpointActivatorDeactivator should be created only if the interval is not max value
if (chkConfig.getCheckpointInterval() != Long.MAX_VALUE) {
    // the periodic checkpoint scheduler is activated and deactivated as a result of
    // job status changes (running -> on, all other states -> off)
    registerJobStatusListener(checkpointCoordinator.createActivatorDeactivator());
}
```
这里的JobStatusListener就是CheckpointCoordinatorDeActivator
```
public JobStatusListener createActivatorDeactivator() {
    synchronized (lock) {
        if (shutdown) {
            throw new IllegalArgumentException("Checkpoint coordinator is shut down");
        }

        if (jobStatusListener == null) {
            jobStatusListener = new CheckpointCoordinatorDeActivator(this);
        }

        return jobStatusListener;
    }
}
```
当监听到状态为RUNNING时会，起动checkpoint定期调度线程
```
public class CheckpointCoordinatorDeActivator implements JobStatusListener {

	private final CheckpointCoordinator coordinator;

	public CheckpointCoordinatorDeActivator(CheckpointCoordinator coordinator) {
		this.coordinator = checkNotNull(coordinator);
	}

	@Override
	public void jobStatusChanges(JobID jobId, JobStatus newJobStatus, long timestamp, Throwable error) {
		if (newJobStatus == JobStatus.RUNNING) {
			// start the checkpoint scheduler
			coordinator.startCheckpointScheduler();//起动CheckpointScheduler
		} else {
			// anything else should stop the trigger for now
			coordinator.stopCheckpointScheduler();
		}
	}
}
//CheckpointCoordinator
public void startCheckpointScheduler() {
    synchronized (lock) {
        if (shutdown) {
            throw new IllegalArgumentException("Checkpoint coordinator is shut down");
        }

        // make sure all prior timers are cancelled
        stopCheckpointScheduler();

        periodicScheduling = true;
        currentPeriodicTrigger = scheduleTriggerWithDelay(getRandomInitDelay());
    }
}
//CheckpointCoordinator
private ScheduledFuture<?> scheduleTriggerWithDelay(long initDelay) {
    return timer.scheduleAtFixedRate(
        new ScheduledTrigger(),
        initDelay, baseInterval, TimeUnit.MILLISECONDS);//定期调度任务
}

```
ScheduledTrigger任务线程
```
private final class ScheduledTrigger implements Runnable {

    @Override
    public void run() {
        try {
            triggerCheckpoint(true);
        }
        catch (Exception e) {
            LOG.error("Exception while triggering checkpoint for job {}.", job, e);
        }
    }
}
```
//一些同名方法,然后调用startTriggeringCheckpoint
```
public CompletableFuture<CompletedCheckpoint> triggerCheckpoint(boolean isPeriodic) {
    return triggerCheckpoint(checkpointProperties, null, isPeriodic, false);
}

@VisibleForTesting
public CompletableFuture<CompletedCheckpoint> triggerCheckpoint(
        CheckpointProperties props,
        @Nullable String externalSavepointLocation,
        boolean isPeriodic,
        boolean advanceToEndOfTime) {

    if (advanceToEndOfTime && !(props.isSynchronous() && props.isSavepoint())) {
        return FutureUtils.completedExceptionally(new IllegalArgumentException(
            "Only synchronous savepoints are allowed to advance the watermark to MAX."));
    }

    CheckpointTriggerRequest request = new CheckpointTriggerRequest(props, externalSavepointLocation, isPeriodic, advanceToEndOfTime);
    requestDecider
        .chooseRequestToExecute(request, isTriggering, lastCheckpointCompletionRelativeTime)
        .ifPresent(this::startTriggeringCheckpoint);
    return request.onCompletionPromise;
}
```

```
//CheckpointCoordinator
final long checkpointId = checkpoint.getCheckpointId();
snapshotTaskState(
	timestamp,
	checkpointId,
	checkpoint.getCheckpointStorageLocation(),
	request.props,
	executions,
	request.advanceToEndOfTime);
//CheckpointCoordinator
private void snapshotTaskState(
	long timestamp,
	long checkpointID,
	CheckpointStorageLocation checkpointStorageLocation,
	CheckpointProperties props,
	Execution[] executions,
	boolean advanceToEndOfTime) {

	final CheckpointOptions checkpointOptions = new CheckpointOptions(
		props.getCheckpointType(),
		checkpointStorageLocation.getLocationReference(),
		isExactlyOnceMode,
		props.getCheckpointType() == CheckpointType.CHECKPOINT && unalignedCheckpointsEnabled);

	// send the messages to the tasks that trigger their checkpoint
	for (Execution execution: executions) {
		if (props.isSynchronous()) {
			execution.triggerSynchronousSavepoint(checkpointID, timestamp, checkpointOptions, advanceToEndOfTime);
		} else {
			execution.triggerCheckpoint(checkpointID, timestamp, checkpointOptions);
		}
	}
}
```
Execution.triggerCheckpoint-->Execution.triggerCheckpointHelper-->taskManagerGateway.triggerCheckpoint-->TaskManagerGateway.triggerCheckpoint-->taskExecutorGateway.triggerCheckpoint()

这里taskExecutorGateway的实现类有TaskExecutor，在TaskExecutor的triggerCheckpoint方法里看到了triggerCheckpointBarrier

//TaskExecutor triggerCheckpointBarrier

```
public CompletableFuture<Acknowledge> triggerCheckpoint(
    ExecutionAttemptID executionAttemptID,
    long checkpointId,
    long checkpointTimestamp,
    CheckpointOptions checkpointOptions,
    boolean advanceToEndOfEventTime) {
  log.debug("Trigger checkpoint {}@{} for {}.", checkpointId, checkpointTimestamp, executionAttemptID);

  final CheckpointType checkpointType = checkpointOptions.getCheckpointType();
  if (advanceToEndOfEventTime && !(checkpointType.isSynchronous() && checkpointType.isSavepoint())) {
    throw new IllegalArgumentException("Only synchronous savepoints are allowed to advance the watermark to MAX.");
  }

  final Task task = taskSlotTable.getTask(executionAttemptID);

  if (task != null) {
    task.triggerCheckpointBarrier(checkpointId, checkpointTimestamp, checkpointOptions, advanceToEndOfEventTime);

    return CompletableFuture.completedFuture(Acknowledge.get());
  } else {
    final String message = "TaskManager received a checkpoint request for unknown task " + executionAttemptID + '.';

    log.debug(message);
    return FutureUtils.completedExceptionally(new CheckpointException(message, CheckpointFailureReason.TASK_CHECKPOINT_FAILURE));
  }
}
//
if (executionState == ExecutionState.RUNNING && invokable != null) {
	try {
		invokable.triggerCheckpointAsync(checkpointMetaData, checkpointOptions, advanceToEndOfEventTime);
	}
	catch (RejectedExecutionException ex) {}
}
```
这里的invokable可以是StreamTask，SourceStreamTask，这两其实最终调用的是一个，因为子类调用super.triggerCheckpointAsync
主要流程

1. -->SourceStreamTask.triggerCheckpointAsync-->StreamTask.triggerCheckpointAsyn-->StreamTask.triggerCheckpoint
2. -->StreamTask.performCheckpoint-->subtaskCheckpointCoordinator.checkpointState-->operatorChain.prepareSnapshotPreBarrier
3. -->operatorChain.broadcastEvent(CheckpointBarrier)-->subtaskCheckpointCoordinator-->takeSnapshotSync
4. -->operatorChain.snapshotState()-->subtaskCheckpointCoordinator.finishAndReportAsync
5. -->AsyncCheckpointRunnable.reportCompletedSnapshotStates--> TaskStateManagerImpl.reportTaskStateSnapshots
6. -->checkpointResponder.acknowledgeCheckpoint-->checkpointCoordinatorGateway.acknowledgeCheckpoint
7. -->JobMaster.acknowledgeCheckpoint-->executionGraphHandler.acknowledgeCheckpoint
8. -->coordinator.receiveAcknowledgeMessage-->coordinator.completePendingCheckpoint
9. -->coordinator.sendAcknowledgeMessages-->ee.notifyCheckpointComplete--> coordinatorContext.notifyCheckpointComplete(checkpointId);

最新版的都抽到SubtaskCheckpointCoordinator里了

```
public void checkpointState(
		CheckpointMetaData metadata,
		CheckpointOptions options,
		CheckpointMetrics metrics,
		OperatorChain<?, ?> operatorChain,
		Supplier<Boolean> isCanceled) throws Exception {

	checkNotNull(options);
	checkNotNull(metrics);

	// All of the following steps happen as an atomic step from the perspective of barriers and
	// records/watermarks/timers/callbacks.
	// We generally try to emit the checkpoint barrier as soon as possible to not affect downstream
	// checkpoint alignments

	if (lastCheckpointId >= metadata.getCheckpointId()) {
		LOG.info("Out of order checkpoint barrier (aborted previously?): {} >= {}", lastCheckpointId, metadata.getCheckpointId());
		channelStateWriter.abort(metadata.getCheckpointId(), new CancellationException(), true);
		checkAndClearAbortedStatus(metadata.getCheckpointId());
		return;
	}

	// Step (0): Record the last triggered checkpointId and abort the sync phase of checkpoint if necessary.
	lastCheckpointId = metadata.getCheckpointId();
	if (checkAndClearAbortedStatus(metadata.getCheckpointId())) {
		// broadcast cancel checkpoint marker to avoid downstream back-pressure due to checkpoint barrier align.
		operatorChain.broadcastEvent(new CancelCheckpointMarker(metadata.getCheckpointId()));
		LOG.info("Checkpoint {} has been notified as aborted, would not trigger any checkpoint.", metadata.getCheckpointId());
		return;
	}

	// Step (1): Prepare the checkpoint, allow operators to do some pre-barrier work.
	//           The pre-barrier work should be nothing or minimal in the common case.
	operatorChain.prepareSnapshotPreBarrier(metadata.getCheckpointId());

	// Step (2): Send the checkpoint barrier downstream
	operatorChain.broadcastEvent(
		new CheckpointBarrier(metadata.getCheckpointId(), metadata.getTimestamp(), options),
		options.isUnalignedCheckpoint());

	// Step (3): Prepare to spill the in-flight buffers for input and output
	if (options.isUnalignedCheckpoint()) {
		prepareInflightDataSnapshot(metadata.getCheckpointId());
	}

	// Step (4): Take the state snapshot. This should be largely asynchronous, to not impact progress of the
	// streaming topology

	Map<OperatorID, OperatorSnapshotFutures> snapshotFutures = new HashMap<>(operatorChain.getNumberOfOperators());
	try {
		if (takeSnapshotSync(snapshotFutures, metadata, metrics, options, operatorChain, isCanceled)) {
			finishAndReportAsync(snapshotFutures, metadata, metrics, options);
		} else {
			cleanup(snapshotFutures, metadata, metrics, new Exception("Checkpoint declined"));
		}
	} catch (Exception ex) {
		cleanup(snapshotFutures, metadata, metrics, ex);
		throw ex;
	}
}
```

