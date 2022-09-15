---
title: dolphinscheduler调度系统源码
tagline: ""
category : Java
layout: post
tags : [Java, File, tools，OOM]
---
## 1.架构设计
1.老版本master将任务直接写入zk,Work通过抢占式从zk获取任务，work与master无直接通信交互   
2.新版master与work直接通信，master通过rpc远程调用将任务下发给work执行

### 1.1 老版(1.2.1为例)架构
![1.2.1设计](https://raw.githubusercontent.com/2pc/2pc.github.io/master/_posts/images/ds2.png)


### 1.2 新版(2.x github dev)架构  
![1.3设计](https://raw.githubusercontent.com/2pc/2pc.github.io/master/_posts/images/ds1.png)

## 2.任务分发
### 2.1 老版任务分发
#### 2.1.1 master将任务提交给zk
```
//ProcessDao.java
public Boolean submitTaskToQueue(TaskInstance taskInstance) {

    try{
        if(taskInstance.isSubProcess()){
            return true;
        }
        if(taskInstance.getState().typeIsFinished()){
            logger.info(String.format("submit to task queue, but task [%s] state [%s] is already  finished. ", taskInstance.getName(), taskInstance.getState().toString()));
            return true;
        }
        // task cannot submit when running
        if(taskInstance.getState() == ExecutionStatus.RUNNING_EXEUTION){
            logger.info(String.format("submit to task queue, but task [%s] state already be running. ", taskInstance.getName()));
            return true;
        }
        if(checkTaskExistsInTaskQueue(taskInstance)){
            logger.info(String.format("submit to task queue, but task [%s] already exists in the queue.", taskInstance.getName()));
            return true;
        }
        logger.info("task ready to queue: {}" , taskInstance);
        boolean insertQueueResult = taskQueue.add(DOLPHINSCHEDULER_TASKS_QUEUE, taskZkInfo(taskInstance));
        logger.info(String.format("master insert into queue success, task : %s", taskInstance.getName()) );
        return insertQueueResult;
    }catch (Exception e){
        logger.error("submit task to queue Exception: ", e);
        logger.error("task queue error : %s", JSONUtils.toJson(taskInstance));
        return false;
    }
}
//TaskQueueZkImpl.java
@Override
public boolean add(String key, String value){
    try {
        String taskIdPath = getTasksPath(key) + Constants.SINGLE_SLASH + value;
        zookeeperOperator.persist(taskIdPath, value);
        return true;
    } catch (Exception e) {
        logger.error("add task to tasks queue exception",e);
        return false;
    }

}
```
#### 2.1.2 work从zk获取任务
```
//FetchTaskThread.java
@Override
public void run() {
    logger.info("worker start fetch tasks...");
    while (Stopper.isRunning()){
        InterProcessMutex mutex = null;
        String currentTaskQueueStr = null;
        try {
            //......省略部分代码......
            //whether have tasks, if no tasks , no need lock  //get all tasks
            List<String> tasksQueueList = taskQueue.getAllTasks(Constants.DOLPHINSCHEDULER_TASKS_QUEUE);
            if (CollectionUtils.isEmpty(tasksQueueList)){
                Thread.sleep(Constants.SLEEP_TIME_MILLIS);
                continue;
            }
            // creating distributed locks, lock path /dolphinscheduler/lock/worker
            mutex = zkWorkerClient.acquireZkLock(zkWorkerClient.getZkClient(),
                    zkWorkerClient.getWorkerLockPath());


            // task instance id str
            List<String> taskQueueStrArr = taskQueue.poll(Constants.DOLPHINSCHEDULER_TASKS_QUEUE, taskNum);
            //......省略部分代码......
                // submit task
                workerExecService.submit(new TaskScheduleThread(taskInstance, processDao));

                // remove node from zk
                removeNodeFromTaskQueue(taskQueueStr);
            }

        }catch (Exception e){
            processErrorTask(currentTaskQueueStr);
            logger.error("fetch task thread failure" ,e);
        }finally {
            AbstractZKClient.releaseMutex(mutex);
        }
    }
}
//TaskQueueZkImpl
@Override
public List<String> poll(String key, int tasksNum) {
    try{
        List<String> list = zookeeperOperator.getChildrenKeys(getTasksPath(key));

        if(list != null && list.size() > 0){}
//......省略部分代码......
}
```

### 2.2 新版任务分发
#### 2.2.1  master任务分发
```
//CommonTaskProcessor.java
public boolean dispatchTask() {
    try {
        if (taskUpdateQueue == null) {
            this.initQueue();
        }
        if (taskInstance.getState().typeIsFinished()) {
            logger.info(String.format("submit task , but task [%s] state [%s] is already  finished. ", taskInstance.getName(), taskInstance.getState().toString()));
            return true;
        }
        // task cannot be submitted because its execution state is RUNNING or DELAY.
        if (taskInstance.getState() == ExecutionStatus.RUNNING_EXECUTION
                || taskInstance.getState() == ExecutionStatus.DELAY_EXECUTION) {
            logger.info("submit task, but the status of the task {} is already running or delayed.", taskInstance.getName());
            return true;
        }
        logger.info("task ready to submit: {}", taskInstance);

        TaskPriority taskPriority = new TaskPriority(processInstance.getProcessInstancePriority().getCode(),
                processInstance.getId(), taskInstance.getProcessInstancePriority().getCode(),
                taskInstance.getId(), org.apache.dolphinscheduler.common.Constants.DEFAULT_WORKER_GROUP);

        TaskExecutionContext taskExecutionContext = getTaskExecutionContext(taskInstance);
        taskPriority.setTaskExecutionContext(taskExecutionContext);

        taskUpdateQueue.put(taskPriority);
        logger.info(String.format("master submit success, task : %s", taskInstance.getName()));
        return true;
    } catch (Exception e) {
        logger.error("submit task error", e);
        return false;
    }
}
//TaskPriorityQueueImpl
@Override
public void put(TaskPriority taskPriorityInfo) throws TaskPriorityQueueException {
    queue.put(taskPriorityInfo);
}
//TaskPriorityQueueConsumer
private List<TaskPriority> batchDispatch(int fetchTaskNum) throws TaskPriorityQueueException, InterruptedException {
    List<TaskPriority> failedDispatchTasks = new ArrayList<>();
    CountDownLatch latch = new CountDownLatch(fetchTaskNum);

    for (int i = 0; i < fetchTaskNum; i++) {
        TaskPriority taskPriority = taskPriorityQueue.poll(Constants.SLEEP_TIME_MILLIS, TimeUnit.MILLISECONDS);
        if (Objects.isNull(taskPriority)) {
            latch.countDown();
            continue;
        }

        consumerThreadPoolExecutor.submit(() -> {
            boolean dispatchResult = this.dispatchTask(taskPriority);
            if (!dispatchResult) {
                failedDispatchTasks.add(taskPriority);
            }
            latch.countDown();
        });
    }

    latch.await();

    return failedDispatchTasks;
}
//TaskPriorityQueueImpl
protected boolean dispatchTask(TaskPriority taskPriority) {
    boolean result = false;
    try {
        TaskExecutionContext context = taskPriority.getTaskExecutionContext();
        ExecutionContext executionContext = new ExecutionContext(context.toCommand(), ExecutorType.WORKER, context.getWorkerGroup());

        if (isTaskNeedToCheck(taskPriority)) {
            if (taskInstanceIsFinalState(taskPriority.getTaskId())) {
                // when task finish, ignore this task, there is no need to dispatch anymore
                return true;
            }
        }

        result = dispatcher.dispatch(executionContext);
    } catch (ExecuteException e) {
        logger.error("dispatch error: {}", e.getMessage(), e);
    }
    return result;
}
```
再看下这个dispatcher(ExecutorDispatcher),最终会选个一个host通过nettyRemotingClient发送给远程的work执行任务
```
public Boolean dispatch(final ExecutionContext context) throws ExecuteException {
    /**
     * get executor manager
     */
    ExecutorManager<Boolean> executorManager = this.executorManagers.get(context.getExecutorType());
    if(executorManager == null){
        throw new ExecuteException("no ExecutorManager for type : " + context.getExecutorType());
    }

    /**
     * host select
     */

    Host host = hostManager.select(context);
    if (StringUtils.isEmpty(host.getAddress())) {
        throw new ExecuteException(String.format("fail to execute : %s due to no suitable worker, "
                        + "current task needs worker group %s to execute",
                context.getCommand(),context.getWorkerGroup()));
    }
    context.setHost(host);
    executorManager.beforeExecute(context);
    try {
        /**
         * task execute
         */
        return executorManager.execute(context);
    } finally {
        executorManager.afterExecute(context);
    }
}

```
#### 2.2.2  work获取执行任务
新版的work端处理倒是挺简单的,直接看TaskExecuteProcessor就好了，毕竟再网上一层就是netty的处理机制了,NettyServerHandler
```
//TaskExecuteProcessor.java
public void process(Channel channel, Command command) {
   //......省略部分代码......
    this.doAck(taskExecutionContext);

    // submit task to manager
    if (!workerManager.offer(new TaskExecuteThread(taskExecutionContext, taskCallbackService, alertClientService, taskPluginManager))) {
        logger.info("submit task to manager error, queue is full, queue size is {}", workerManager.getDelayQueueSize());
    }
}
//WorkerManagerThread.java
public boolean offer(TaskExecuteThread taskExecuteThread) {
    return workerExecuteQueue.offer(taskExecuteThread);
}
//WorkerManagerThread.java
public void run() {
    Thread.currentThread().setName("Worker-Execute-Manager-Thread");
    TaskExecuteThread taskExecuteThread;
    while (Stopper.isRunning()) {
        try {
            taskExecuteThread = workerExecuteQueue.take();
            workerExecService.submit(taskExecuteThread);
        } catch (Exception e) {
            logger.error("An unexpected interrupt is happened, "
                + "the exception will be ignored and this thread will continue to run", e);
        }
    }
}
```
看下TaskExecuteThread，关键也是run()方法
```
public void run() {
   //......省略部分代码......
    task = taskChannel.createTask(taskRequest);

    // task init
    this.task.init();

    //init varPool
    this.task.getParameters().setVarPool(taskExecutionContext.getVarPool());

    // task handle
    this.task.handle();

    // task result process
    if (this.task.getNeedAlert()) {
        sendAlert(this.task.getTaskAlertInfo());
    }
       //......省略部分代码......
}
这里的task已经是最终的任务，主要实现有ShellTask,SqlTask,HttpTask,PythonTask,DataXTask,SeaTunnelTask等




## link
[图片来源](https://mp.weixin.qq.com/s/xHviN-2KW7Jwgq5hqhwQSQ)
