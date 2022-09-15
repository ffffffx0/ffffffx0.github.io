---
layout: post
title: "Zookeeper"
keywords: ["distributed"]
description: "Paxos协议"
category: "distributed"
tags: ["distributed","Paxos","Zookeeper"]
---

### Zookeepeer 通信以及心跳机制

### Zookeepeer Leader 选举

#### 如何确定两张选票的大小

注释已经写的很清楚了

>
1. 选举轮数epoch的比较，这个大的，选票就大
2. 选举轮数相同的话，比较(事务号)zxid，事务号大，选票也大
3. 选举轮数，事务号都一样，比较节点的id,id大的选票也大


```
protected boolean totalOrderPredicate(long newId, long newZxid, long newEpoch, long curId, long curZxid, long curEpoch) {
    LOG.debug("id: " + newId + ", proposed id: " + curId + ", zxid: 0x" +
            Long.toHexString(newZxid) + ", proposed zxid: 0x" + Long.toHexString(curZxid));
    if(self.getQuorumVerifier().getWeight(newId) == 0){
        return false;
    }
    
    /*
     * We return true if one of the following three cases hold:
     * 1- New epoch is higher
     * 2- New epoch is the same as current epoch, but new zxid is higher
     * 3- New epoch is the same as current epoch, new zxid is the same
     *  as current zxid, but server id is higher.
     */
    
    return ((newEpoch > curEpoch) || 
            ((newEpoch == curEpoch) &&
            ((newZxid > curZxid) || ((newZxid == curZxid) && (newId > curId)))));
}
```

####  LOOKING 状态时开启一轮新的选举

首先是选举的轮数(electionEpoch/logicalclock)加1，并将选票更新为自己广播出去。

```
synchronized(this){
	//electionEpoch 轮数增1
    logicalclock++;
    //投票给自己（proposedLeader = leader（sid）; proposedZxid = zxid;proposedEpoch = epoch;）
    updateProposal(getInitId(), getInitLastLoggedZxid(), getPeerEpoch());//sid,zxid,epoch
}

LOG.info("New election. My id =  " + self.getId() +
        ", proposed zxid=0x" + Long.toHexString(proposedZxid));
//发送投票，构造ToSend（this.electionEpoch = logicalclock）
sendNotifications();

```

另一方面，会接受其他server 过来的投票，收到投票头进行选票的比较

##### 如果对方选举轮数比较靠前

>
1. 需要更新逻辑时钟（选举轮数），
2. 清空收票箱（都是上一轮的投票）
3. 因为现在选举轮数又一样了，所以需要再次PK下选票，再广播出去

```
// If notification > current, replace and send messages out
// 对方选举轮数靠前比自己大，更新选举轮数（逻辑时钟），清空收票箱（都是上一轮的投票），
// 再比较（比较proposed）投递过来的与自己的比较，如果对方大了，更新为新的投票，不然还是投给自己
if (n.electionEpoch > logicalclock) {
    logicalclock = n.electionEpoch;
    recvset.clear();
    if(totalOrderPredicate(n.leader, n.zxid, n.peerEpoch,
            getInitId(), getInitLastLoggedZxid(), getPeerEpoch())) {
        updateProposal(n.leader, n.zxid, n.peerEpoch);
    } else {
        updateProposal(getInitId(),
                getInitLastLoggedZxid(),
                getPeerEpoch());
    }
    sendNotifications();
```

##### 如果当前自己的选举轮数靠前了，那还用比嘛？

```
else if (n.electionEpoch < logicalclock) {
    if(LOG.isDebugEnabled()){
        LOG.debug("Notification election epoch is smaller than logicalclock. n.electionEpoch = 0x"
                + Long.toHexString(n.electionEpoch)
                + ", logicalclock=0x" + Long.toHexString(logicalclock));
    }
    break;
} 
```

##### 选举轮数一致，ok 那就pk下，看谁胜出

```
//选举轮数一致，比较选票，将胜出的更新为新的选票并广播出去
} else if (totalOrderPredicate(n.leader, n.zxid, n.peerEpoch,
        proposedLeader, proposedZxid, proposedEpoch)) {
    updateProposal(n.leader, n.zxid, n.peerEpoch);
    sendNotifications();
}
```



### Zookeepeer 读写流程

#### Read一致性保证   

>
1. 尽管zookeeper保证了大多数，但是如果client读取到这些大多数以外的节点,必然会读取到老数据，zookeeper可以通过执行sync来解决   
2. watch机制可否？A,B均注册watcher到节点，当A更新节点数据时，server通知B，B执行watcher


[ZooKeeper-Consistency-Guarantees](https://phoenixjiangnan.github.io/2016/07/04/distributed%20system/zookeeper/ZooKeeper-Consistency-Guarantees/)

#### Zookeeper之分布式系统中生成全局唯一ID-SessionId

>
1. 64位，其中高8位表示机器序列号，即所在的机器，后56为用当前时间的毫秒数随机
2. 3.4.6之前使用右移操作，3.4.6使用无符号右移。避免负数导致无法区分机器序列号
3. 3.5.0开始不实用毫秒数，而是使用了System.nanoTime避免人为改动系统时间可能导致的问题

##### 3.4.5(3.4.6之前?)

```
public static long initializeNextSession(long id) {
    long nextSid = 0;
    nextSid = (System.currentTimeMillis() << 24) >> 8;
    nextSid =  nextSid | (id <<56);
    return nextSid;
}
```

##### 3.4.6

```
public static long initializeNextSession(long id) {
    long nextSid = 0;
    nextSid = (System.currentTimeMillis() << 24) >>> 8;
    nextSid =  nextSid | (id <<56);
    return nextSid;
}
```

##### 3.5.0

```
public static long initializeNextSession(long id) {
    long nextSid;
    nextSid = (Time.currentElapsedTime() << 24) >>> 8;
    nextSid =  nextSid | (id <<56);
    return nextSid;
}
/**
* Returns time in milliseconds as does System.currentTimeMillis(),
* but uses elapsed time from an arbitrary epoch more like System.nanoTime().
* The difference is that if somebody changes the system clock,
* Time.currentElapsedTime will change but nanoTime won't. On the other hand,
* all of ZK assumes that time is measured in milliseconds.
* @return  The time in milliseconds from some arbitrary point in time.
*/
public static long currentElapsedTime() {
    return System.nanoTime() / 1000000;
}
```
