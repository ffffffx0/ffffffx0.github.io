---
layout: post
title: "SolrCloud Leader Elect"
keywords: ["distributed","Search","SolrCloud"]
description: "SolrCloud Leader Elect"
category: "distributed"
tags: ["distributed","Search","SolrCloud"]
---
SolrCloud 领导选举，最小seq,这点有点同Kafka seq(0)

主要有两部分，一个是集群的leader选举过程，一个是collection的shard的leader选举过程

### 集群的leader选举

>
1. 主要相关的类：OverseerElectionContext
2. 相关节点：/overseer_elect/election可以说leader候选节点，/overseer_elect/election是选举出来的leader

```
[zk: 172.16.113.253:2182,172.16.113.254:2182
(CONNECTED) 56] ls /overseer_elect/election
[166405419031556844-172.17.52.154:8080_solr-n_0000000117, 166405419031556837-172.17.52.155:8080_solr-n_0000000115]
[zk: 172.16.113.253:2182,172.16.113.254:2182(CONNECTED) 57] get  /overseer_elect/election
null
cZxid = 0x36002c1fe2
ctime = Fri May 06 06:02:06 EDT 2016
mZxid = 0x36002c1fe2
mtime = Fri May 06 06:02:06 EDT 2016
pZxid = 0x36002d5efe
cversion = 234
dataVersion = 0
aclVersion = 0
ephemeralOwner = 0x0
dataLength = 0
numChildren = 2
```

集群leader的大致流程调用关系SolrDispatchFilter.init()-->SolrDispatchFilter.createCoreContainer()-->CoreContainer.load()-->ZkContainer.initZooKeeper()-->new ZkController()-->new SolrZkClient()与ZkController.init()中都有选举相关代码

在SolrZkClient()的主要代码

```
///overseer_elect/election下seq最小的节点为leader
ElectionContext context = new OverseerElectionContext(zkClient,
    overseer, getNodeName());

ElectionContext prevContext = overseerElector.getContext();
if (prevContext != null) {
  prevContext.cancelElection();
}

overseerElector.setup(context);
//选出seq最小的节点作为leader
overseerElector.joinElection(context, true);
```
在ZkController.init()的主要代码

```
if (!zkRunOnly) {
      overseerElector = new LeaderElector(zkClient);
      this.overseer = new Overseer(shardHandler, updateShardHandler,
          adminPath, zkStateReader, this, cc.getConfig());
      ElectionContext context = new OverseerElectionContext(zkClient,
          overseer, getNodeName());
      overseerElector.setup(context);
      overseerElector.joinElection(context, false);
    }
```

看起来都差不多，主要是两行代码，overseerElector.setup(context)也没干啥事，就是确保/overseer_elect/election节点存在，有就直接返回，没有就创建

```
overseerElector.setup(context);
overseerElector.joinElection(context, false);
```

主要看joinElection(ElectionContext context, boolean replacement)，逻辑主要在joinElection(ElectionContext context, boolean replacement,boolean joinAtHead)中实现

```
 /**
   * Begin participating in the election process. Gets a new sequential number
   * and begins watching the node with the sequence number before it, unless it
   * is the lowest number, in which case, initiates the leader process. If the
   * node that is watched goes down, check if we are the new lowest node, else
   * watch the next lowest numbered node.
   *
   * @return sequential node number
   */
public int joinElection(ElectionContext context, boolean replacement,boolean joinAtHead) throws KeeperException, InterruptedException, IOException {
  context.joinedElectionFired();
  
  final String shardsElectZkPath = context.electionPath + LeaderElector.ELECTION_NODE;
  
  long sessionId = zkClient.getSolrZooKeeper().getSessionId();
  String id = sessionId + "-" + context.id;
  String leaderSeqPath = null;
  boolean cont = true;
  int tries = 0;
  while (cont) {
    try {
      if(joinAtHead){
        log.info("node {} Trying to join election at the head ", id);
        /**
         * "/overseer_elect/election"中序号最小的节点为leader
         * 类似n_0000000001 or n_0000000003中较小的节点
         */
        List<String> nodes = OverseerCollectionProcessor.getSortedElectionNodes(zkClient);
        if(nodes.size() <2){
          leaderSeqPath = zkClient.create(shardsElectZkPath + "/" + id + "-n_", null,
              CreateMode.EPHEMERAL_SEQUENTIAL, false);
        } else {
          String firstInLine = nodes.get(1);
          log.info("The current head: {}", firstInLine);
          Matcher m = LEADER_SEQ.matcher(firstInLine);
          if (!m.matches()) {
            throw new IllegalStateException("Could not find regex match in:"
                + firstInLine);
          }
          leaderSeqPath = shardsElectZkPath + "/" + id + "-n_"+ m.group(1);
          zkClient.create(leaderSeqPath, null, CreateMode.EPHEMERAL, false);
          log.info("Joined at the head  {}", leaderSeqPath );

        }
      } else {
        leaderSeqPath = zkClient.create(shardsElectZkPath + "/" + id + "-n_", null,
            CreateMode.EPHEMERAL_SEQUENTIAL, false);
      }

      context.leaderSeqPath = leaderSeqPath;
      cont = false;
    } catch (ConnectionLossException e) {
      // we don't know if we made our node or not...
      List<String> entries = zkClient.getChildren(shardsElectZkPath, null, true);
      
      boolean foundId = false;
      for (String entry : entries) {
        String nodeId = getNodeId(entry);
        if (id.equals(nodeId)) {
          // we did create our node...
          foundId  = true;
          break;
        }
      }
      if (!foundId) {
        cont = true;
        if (tries++ > 20) {
          throw new ZooKeeperException(SolrException.ErrorCode.SERVER_ERROR,
              "", e);
        }
        try {
          Thread.sleep(50);
        } catch (InterruptedException e2) {
          Thread.currentThread().interrupt();
        }
      }

    } catch (KeeperException.NoNodeException e) {
      // we must have failed in creating the election node - someone else must
      // be working on it, lets try again
      if (tries++ > 20) {
        context = null;
        throw new ZooKeeperException(SolrException.ErrorCode.SERVER_ERROR,
            "", e);
      }
      cont = true;
      try {
        Thread.sleep(50);
      } catch (InterruptedException e2) {
        Thread.currentThread().interrupt();
      }
    }
  }
  int seq = getSeq(leaderSeqPath);
  //
  checkIfIamLeader(seq, context, replacement);
  
  return seq;
}
```

暂且不关注joinAtHead这段,首先创建的节点是(shardsElectZkPath + "/" + id + "-n_"),注意这里-n_后边是没有序列号的(类似/overseer_elect/election/166405419031556844-172.17.52.154:8080_solr-n_，正常的应该是/overseer_elect/election/166405419031556844-172.17.52.154:8080_solr-n_0000000117)，最多尝试20次，这样会导致后边拿到的int行seq为0，自然是最小的？

```
 leaderSeqPath = zkClient.create(shardsElectZkPath + "/" + id + "-n_", null,
            CreateMode.EPHEMERAL_SEQUENTIAL, false);
```
getSeq从n_0000000117的串中取出int序列号，这里的nStringSequence是n_xxxxxxx

```
public static int getSeq(String nStringSequence) {
    int seq = 0;
    Matcher m = LEADER_SEQ.matcher(nStringSequence);
    if (m.matches()) {
      seq = Integer.parseInt(m.group(1));
    } else {
      throw new IllegalStateException("Could not find regex match in:"
          + nStringSequence);
    }
    return seq;
}
```
在checkIfIamLeader()中判断自己是不是leader,首先与最小的比较，如果小于或等于最小的加入选举，不然找到自己的最靠前的位置(就是插入到seqs保持seqs从小到大排列不变)，并插入，并对前一个节点(这里的index为i - 2，因为i就是找到的比自己大的，i-1就是本身)进行监听

```
 /**
   * Check if the candidate with the given n_* sequence number is the leader.
   * If it is, set the leaderId on the leader zk node. If it is not, start
   * watching the candidate that is in line before this one - if it goes down, check
   * if this candidate is the leader again.
   *
   * @param replacement has someone else been the leader already?
   */
  private void checkIfIamLeader(final int seq, final ElectionContext context, boolean replacement) throws KeeperException,
      InterruptedException, IOException {
    context.checkIfIamLeaderFired();
    // get all other numbers...
    final String holdElectionPath = context.electionPath + ELECTION_NODE;
    List<String> seqs = zkClient.getChildren(holdElectionPath, null, true);

    sortSeqs(seqs);
    List<Integer> intSeqs = getSeqs(seqs);
    if (intSeqs.size() == 0) {
      log.warn("Our node is no longer in line to be leader");
      return;
    }
    if (seq <= intSeqs.get(0)) {
      if(seq == intSeqs.get(0) && !context.leaderSeqPath.equals(holdElectionPath+"/"+seqs.get(0)) ) {//somebody else already  became the leader with the same sequence id , not me
        log.info("was going be leader {} , seq(0) {}",context.leaderSeqPath,holdElectionPath+"/"+seqs.get(0));//but someone else jumped the line
        retryElection(context,false);//join at the tail again
        return;
      }
      // first we delete the node advertising the old leader in case the ephem is still there
      // first we delete the node advertising the old leader in case the ephem is still there
      try {
        // 通通先删除再说，后边我会自己注册上去的
        zkClient.delete(context.leaderPath, -1, true);
      }catch (KeeperException.NoNodeException nne){
        //no problem
      }catch (InterruptedException e){
        throw e;
      } catch (Exception e) {
        //failed to delete the leader node
        log.error("leader elect delete error",e);
        retryElection(context, false);
        return;
        // fine
      }

      try {
        //我要自己注册上去了，登基成leader了
        runIamLeaderProcess(context, replacement);
      } catch (KeeperException.NodeExistsException e) {
        log.error("node exists",e);
        retryElection(context, false);
        return;
      }
    } else {
      // I am not the leader - watch the node below me
      int i = 1;
      for (; i < intSeqs.size(); i++) {
        int s = intSeqs.get(i);
        if (seq < s) {
          // we found who we come before - watch the guy in front
          break;
        }
      }
      int index = i - 2;
      if (index < 0) {
        log.warn("Our node is no longer in line to be leader");
        return;
      }
      try {
        String watchedNode = holdElectionPath + "/" + seqs.get(index);
        zkClient.getData(watchedNode, watcher = new ElectionWatcher(context.leaderSeqPath , watchedNode,seq, context) , null, true);
      } catch (KeeperException.SessionExpiredException e) {
        throw e;
      } catch (KeeperException e) {
        log.warn("Failed setting watch", e);
        // we couldn't set our watch - the node before us may already be down?
        // we need to check if we are the leader again
        checkIfIamLeader(seq, context, true);
      }
    }
  }
```
判定自己为leader后,在runIamLeaderProcess里调用OverseerElectionContext.runLeaderProcess()需要将节点id(如：{"id":"166405419031556837-172.17.52.155:8080_solr-n_0000000115"})到/overseer_elect/leader，

```
void runLeaderProcess(boolean weAreReplacement, int pauseBeforeStartMs) throws KeeperException,
  InterruptedException {
    log.info("I am going to be the leader {}", id);
    final String id = leaderSeqPath
        .substring(leaderSeqPath.lastIndexOf("/") + 1);
    ZkNodeProps myProps = new ZkNodeProps("id", id);
    
    zkClient.makePath(leaderPath, ZkStateReader.toJSON(myProps),
        CreateMode.EPHEMERAL, true);
    if(pauseBeforeStartMs >0){
      try {
        Thread.sleep(pauseBeforeStartMs);
      } catch (InterruptedException e) {
        Thread.interrupted();
        log.warn("Wait interrupted ", e);
      }
    }
    
    overseer.start(id);
}
```



### collection的shard的leader选举
>
1. 主要相关的类：ShardLeaderElectionContext
2. 相关节点：/collections/collection/leader_elect/shard1/election(shardN表示这个collection得shard)
可以说某个shard对应的leader候选节点，/collections/collection/leaders/shard1是选举出来的leader

```
[zk: 172.16.113.253:2182,172.16.113.254:2182(CONNECTED) 8] ls   /collections/song/leader_elect/shard1/election
[166405419031556844-core_node1-n_0000000028, 166405419031556837-core_node2-n_0000000026]
[zk: 172.16.113.253:2182,172.16.113.254:2182(CONNECTED) 9] get  /collections/song/leaders/shard1              
{
  "core":"song_shard1_replica1",
  "node_name":"172.17.52.155:8080_solr",
  "base_url":"http://172.17.52.155:8080/solr"}
cZxid = 0x36002d5ea2
ctime = Tue Sep 06 05:18:13 EDT 2016
mZxid = 0x36002d5ea2
mtime = Tue Sep 06 05:18:13 EDT 2016
pZxid = 0x36002d5ea2
cversion = 0
dataVersion = 0
aclVersion = 0
ephemeralOwner = 0x24f30d8d21372e5
dataLength = 125
numChildren = 0
```

Shard的leader选举的大致流程调用关系SolrDispatchFilter.init()-->SolrDispatchFilter.createCoreContainer()-->CoreContainer.load()-->CoreContainer.create()-->CoreContainer.registerCore-->ZkContainer.registerInZk()-->ZkController.register-->ZkController.joinElection 

ZkController.joinElection中

```
  private void joinElection(CoreDescriptor cd, boolean afterExpiration) throws InterruptedException, KeeperException, IOException {
    // look for old context - if we find it, cancel it
    String collection = cd.getCloudDescriptor().getCollectionName();
    final String coreNodeName = cd.getCloudDescriptor().getCoreNodeName();
    
    ContextKey contextKey = new ContextKey(collection, coreNodeName);
    
    ElectionContext prevContext = electionContexts.get(contextKey);
    
    if (prevContext != null) {
      prevContext.cancelElection();
    }
    
    String shardId = cd.getCloudDescriptor().getShardId();
    
    Map<String,Object> props = new HashMap<>();
    // we only put a subset of props into the leader node
    props.put(ZkStateReader.BASE_URL_PROP, getBaseUrl());
    props.put(ZkStateReader.CORE_NAME_PROP, cd.getName());
    props.put(ZkStateReader.NODE_NAME_PROP, getNodeName());
    
 
    ZkNodeProps ourProps = new ZkNodeProps(props);

    
    ElectionContext context = new ShardLeaderElectionContext(leaderElector, shardId,
        collection, coreNodeName, ourProps, this, cc);

    leaderElector.setup(context);
    electionContexts.put(contextKey, context);
    leaderElector.joinElection(context, false);
  }
```


选举逻辑也是那两行代码，不过context现在是ShardLeaderElectionContext

```
overseerElector.setup(context);
overseerElector.joinElection(context, false);
```
其他逻辑与集群大致一样，选择/collections/collection/leader_elect/shard1/election下seq最小的最为leader，将leader信息set到/collections/collection/leaders/shard1节点；最后runIamLeaderProcess中调用ShardLeaderElectionContext的runLeaderProcess，这段才是最大的差别

```
context.runLeaderProcess(weAreReplacement,0);
```
```
  @Override
  void runLeaderProcess(boolean weAreReplacement, int pauseBeforeStart) throws KeeperException,
      InterruptedException, IOException {
    log.info("Running the leader process for shard " + shardId);
    
    String coreName = leaderProps.getStr(ZkStateReader.CORE_NAME_PROP);
    
    // clear the leader in clusterstate
    ZkNodeProps m = new ZkNodeProps(Overseer.QUEUE_OPERATION, ZkStateReader.LEADER_PROP,
        ZkStateReader.SHARD_ID_PROP, shardId, ZkStateReader.COLLECTION_PROP,
        collection);
    Overseer.getInQueue(zkClient).offer(ZkStateReader.toJSON(m));
    
    int leaderVoteWait = cc.getZkController().getLeaderVoteWait();
    if (!weAreReplacement) {
      waitForReplicasToComeUp(weAreReplacement, leaderVoteWait);
    }

    try (SolrCore core = cc.getCore(coreName)) {

      if (core == null) {
        cancelElection();
        throw new SolrException(ErrorCode.SERVER_ERROR,
            "Fatal Error, SolrCore not found:" + coreName + " in "
                + cc.getCoreNames());
      }
      
      // should I be leader?
      if (weAreReplacement && !shouldIBeLeader(leaderProps, core, weAreReplacement)) {
        rejoinLeaderElection(leaderSeqPath, core);
        return;
      }
      
      log.info("I may be the new leader - try and sync");
 
      
      // we are going to attempt to be the leader
      // first cancel any current recovery
      core.getUpdateHandler().getSolrCoreState().cancelRecovery();
      
      if (weAreReplacement) {
        // wait a moment for any floating updates to finish
        try {
          Thread.sleep(2500);
        } catch (InterruptedException e) {
          Thread.currentThread().interrupt();
          throw new SolrException(ErrorCode.SERVICE_UNAVAILABLE, e);
        }
      }
      
      boolean success = false;
      try {
        success = syncStrategy.sync(zkController, core, leaderProps, weAreReplacement);
      } catch (Exception e) {
        SolrException.log(log, "Exception while trying to sync", e);
        success = false;
      }
      
      UpdateLog ulog = core.getUpdateHandler().getUpdateLog();

      if (!success) {
        boolean hasRecentUpdates = false;
        if (ulog != null) {
          // TODO: we could optimize this if necessary
          UpdateLog.RecentUpdates recentUpdates = ulog.getRecentUpdates();
          try {
            hasRecentUpdates = !recentUpdates.getVersions(1).isEmpty();
          } finally {
            recentUpdates.close();
          }
        }

        if (!hasRecentUpdates) {
          // we failed sync, but we have no versions - we can't sync in that case
          // - we were active
          // before, so become leader anyway
          log.info("We failed sync, but we have no versions - we can't sync in that case - we were active before, so become leader anyway");
          success = true;
        }
      }
      
      // solrcloud_debug
      if (log.isDebugEnabled()) {
        try {
          RefCounted<SolrIndexSearcher> searchHolder = core
              .getNewestSearcher(false);
          SolrIndexSearcher searcher = searchHolder.get();
          try {
            log.debug(core.getCoreDescriptor().getCoreContainer()
                .getZkController().getNodeName()
                + " synched "
                + searcher.search(new MatchAllDocsQuery(), 1).totalHits);
          } finally {
            searchHolder.decref();
          }
        } catch (Exception e) {
          throw new SolrException(ErrorCode.SERVER_ERROR, null, e);
        }
      }
      if (!success) {
        rejoinLeaderElection(leaderSeqPath, core);
        return;
      }

      log.info("I am the new leader: "
          + ZkCoreNodeProps.getCoreUrl(leaderProps) + " " + shardId);
      core.getCoreDescriptor().getCloudDescriptor().setLeader(true);
    }

    boolean isLeader = true;
    try {
      super.runLeaderProcess(weAreReplacement, 0);
    } catch (Exception e) {
      isLeader = false;
      SolrException.log(log, "There was a problem trying to register as the leader", e);
  
      try (SolrCore core = cc.getCore(coreName)) {

        if (core == null) {
          log.debug("SolrCore not found:" + coreName + " in " + cc.getCoreNames());
          return;
        }
        
        core.getCoreDescriptor().getCloudDescriptor().setLeader(false);
        
        // we could not publish ourselves as leader - try and rejoin election
        rejoinLeaderElection(leaderSeqPath, core);
      }
    }

    if (isLeader) {
      // check for any replicas in my shard that were set to down by the previous leader
      try {
        startLeaderInitiatedRecoveryOnReplicas(coreName);
      } catch (Exception exc) {
        // don't want leader election to fail because of
        // an error trying to tell others to recover
      }
    }    
  }
```
如果选举leader成功还要做相关的Recovery工作，会启动一个LeaderInitiatedRecoveryThread线程

```
startLeaderInitiatedRecoveryOnReplicas(coreName);
```
在LeaderInitiatedRecoveryThread线程中通过HttpSolrServer发送Recovery请求,中间省略部分代码

```
RequestRecovery recoverRequestCmd = new RequestRecovery();
recoverRequestCmd.setAction(CoreAdminAction.REQUESTRECOVERY);
recoverRequestCmd.setCoreName(coreNeedingRecovery);
HttpSolrServer server = new HttpSolrServer(recoveryUrl);
server.setSoTimeout(60000);
server.setConnectionTimeout(15000);
server.request(recoverRequestCmd);
```

在CoreAdminHandler的handleRequestInternal可以找到对应Action的处理逻辑

```
case REQUESTRECOVERY: {
  this.handleRequestRecoveryAction(req, rsp);
  break;
}
```





SolrCloud Group

```
Caused by: java.lang.OutOfMemoryError: Java heap space
        at org.apache.lucene.codecs.blocktree.SegmentTermsEnumFrame.<init>(SegmentTermsEnumFrame.java:53)
        at org.apache.lucene.codecs.blocktree.SegmentTermsEnum.<init>(SegmentTermsEnum.java:78)
        at org.apache.lucene.codecs.blocktree.FieldReader.iterator(FieldReader.java:156)
        at org.apache.lucene.index.ExitableDirectoryReader$ExitableTerms.iterator(ExitableDirectoryReader.java:141)
        at org.apache.lucene.index.TermContext.build(TermContext.java:94)
        at org.apache.lucene.search.TermQuery.createWeight(TermQuery.java:191)
        at org.apache.lucene.search.IndexSearcher.createWeight(IndexSearcher.java:851)
        at org.apache.lucene.search.BooleanWeight.<init>(BooleanWeight.java:57)
        at org.apache.lucene.search.BooleanQuery.createWeight(BooleanQuery.java:184)
        at org.apache.lucene.search.IndexSearcher.createWeight(IndexSearcher.java:851)
        at org.apache.lucene.search.BooleanWeight.<init>(BooleanWeight.java:57)
        at org.apache.lucene.search.BooleanQuery.createWeight(BooleanQuery.java:184)
        at org.apache.lucene.search.FilteredQuery.createWeight(FilteredQuery.java:81)
        at org.apache.lucene.search.IndexSearcher.createWeight(IndexSearcher.java:851)
        at org.apache.lucene.search.IndexSearcher.createNormalizedWeight(IndexSearcher.java:834)
        at org.apache.lucene.search.IndexSearcher.search(IndexSearcher.java:485)
        at org.apache.solr.search.Grouping.searchWithTimeLimiter(Grouping.java:456)
        at org.apache.solr.search.Grouping.execute(Grouping.java:407)
        at org.apache.solr.handler.component.QueryComponent.process(QueryComponent.java:492)
        at org.apache.solr.handler.component.SearchHandler.handleRequestBody(SearchHandler.java:255)
        at org.apache.solr.handler.RequestHandlerBase.handleRequest(RequestHandlerBase.java:143)
        at org.apache.solr.core.SolrCore.execute(SolrCore.java:2064)
        at org.apache.solr.servlet.HttpSolrCall.execute(HttpSolrCall.java:654)
        at org.apache.solr.servlet.HttpSolrCall.call(HttpSolrCall.java:450)
        at org.apache.solr.servlet.SolrDispatchFilter.doFilter(SolrDispatchFilter.java:227)
        at org.apache.solr.servlet.SolrDispatchFilter.doFilter(SolrDispatchFilter.java:196)
        at org.eclipse.jetty.servlet.ServletHandler$CachedChain.doFilter(ServletHandler.java:1652)
        at org.eclipse.jetty.servlet.ServletHandler.doHandle(ServletHandler.java:585)
        at org.eclipse.jetty.server.handler.ScopedHandler.handle(ScopedHandler.java:143)
        at org.eclipse.jetty.security.SecurityHandler.handle(SecurityHandler.java:577)
        at org.eclipse.jetty.server.session.SessionHandler.doHandle(SessionHandler.java:223)
        at org.eclipse.jetty.server.handler.ContextHandler.doHandle(ContextHandler.java:1127)
```

[solr3.5 分布式groupby源码分析](http://blog.sina.com.cn/s/blog_4d58e3c00102v2o1.html)
