---
title: Flink jobmanager&taskmanager的启动
tagline: ""
category : flink
layout: post
tags : [flink, yarn, realtime]
---
YarnApplicationMasterRunner-->run()-->runApplicationMaster()
```
// 2: the JobManager
LOG.debug("Starting JobManager actor");

// we start the JobManager with its standard name
ActorRef jobManager = JobManager.startJobManagerActors(
  config,
  actorSystem,
  futureExecutor,
  ioExecutor,
  highAvailabilityServices,
  metricRegistry,
  webMonitor == null ? Option.empty() : Option.apply(webMonitor.getRestAddress()),
  new Some<>(JobMaster.JOB_MANAGER_NAME),
  Option.<String>empty(),
  getJobManagerClass(),
  getArchivistClass())._1();

// 3: Flink's Yarn ResourceManager
LOG.debug("Starting YARN Flink Resource Manager");

Props resourceMasterProps = YarnFlinkResourceManager.createActorProps(
  getResourceManagerClass(),
  config,
  yarnConfig,
  highAvailabilityServices.getJobManagerLeaderRetriever(HighAvailabilityServices.DEFAULT_JOB_ID),
  appMasterHostname,
  webMonitorURL,
  taskManagerParameters,
  taskManagerContext,
  numInitialTaskManagers,
  LOG);

```
注意YarnFlinkResourceManager，JobManager都是Actor
