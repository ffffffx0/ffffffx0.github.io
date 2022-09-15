---
title: Kafka事务性实现
tagline: ""
category : kafka
layout: post
tags : [kafka]
---
[KIP-98 - Exactly Once Delivery and Transactional Messaging](https://cwiki.apache.org/confluence/display/KAFKA/KIP-98+-+Exactly+Once+Delivery+and+Transactional+Messaging)   
![Transactional Messaging](https://github.com/2pc/2pc.github.io/tree/master/_posts/images/kafka_tx.png)

事务消息示例：
```
// Init transactions call should always happen first in order to clear zombie transactions from previous generation.
//1. 初始事务
producer.initTransactions();
// Begin a new transaction session.
//2. 开始一个事务操作
producer.beginTransaction();
//3. 发送消息
for (ConsumerRecord<Integer, String> record : records) {
    // Process the record and send to downstream.
    ProducerRecord<Integer, String> customizedRecord = transform(record);
    producer.send(customizedRecord);
}
Map<TopicPartition, OffsetAndMetadata> offsets = consumerOffsets();

// Checkpoint the progress by sending offsets to group coordinator broker.
// Note that this API is only available for broker >= 2.5.
producer.sendOffsetsToTransaction(offsets, consumer.groupMetadata());

// Finish the transaction. All sent records should be visible for consumption now.
//commit事务
producer.commitTransaction();
```

### 1. Finding a transaction coordinator -- the FindCoordinatorRequest

### 2. Getting a producer Id -- the InitPidRequest
```
public void initTransactions() {
    throwIfNoTransactionManager();
    throwIfProducerClosed();
    TransactionalRequestResult result = transactionManager.initializeTransactions();
    sender.wakeup();
    result.await(maxBlockTimeMs, TimeUnit.MILLISECONDS);
}
//transactionManager
synchronized TransactionalRequestResult initializeTransactions(ProducerIdAndEpoch producerIdAndEpoch) {
    boolean isEpochBump = producerIdAndEpoch != ProducerIdAndEpoch.NONE;
    return handleCachedTransactionRequestResult(() -> {
        // If this is an epoch bump, we will transition the state as part of handling the EndTxnRequest
        if (!isEpochBump) {
            transitionTo(State.INITIALIZING);
            log.info("Invoking InitProducerId for the first time in order to acquire a producer ID");
        } else {
            log.info("Invoking InitProducerId with current producer ID and epoch {} in order to bump the epoch", producerIdAndEpoch);
        }
        //发送InitProducerIdRequest
        InitProducerIdRequestData requestData = new InitProducerIdRequestData()
                .setTransactionalId(transactionalId)
                .setTransactionTimeoutMs(transactionTimeoutMs)
                .setProducerId(producerIdAndEpoch.producerId)
                .setProducerEpoch(producerIdAndEpoch.epoch);
        InitProducerIdHandler handler = new InitProducerIdHandler(new InitProducerIdRequest.Builder(requestData),
                isEpochBump);
        enqueueRequest(handler);
        return handler.result;
    }, State.INITIALIZING);
    }

```
### 3.  Starting a Transaction – The beginTransaction() API
### 4. The consume-transform-produce loop

#### 4.1 AddPartitionsToTxnRequest

#### 4.2 ProduceRequest

#### 4.3 AddOffsetCommitsToTxnRequest


#### 4.4 TxnOffsetCommitRequest

### 5  Committing or Aborting a Transaction

#### 5.1 EndTxnRequest

#### 5.2 WriteTxnMarkerRequest

#### 5.3 Writing the final Commit or Abort Message

initTransactions 依据transactionId 获取PID与epoch



producer(transactionalId-->PID)

AddPartitionsToTxnRequest

TxnOffsetCommitRequset

[Transactions in Apache Kafka](https://www.confluent.io/blog/transactions-apache-kafka/)
[https://docs.google.com/document/d/1Rlqizmk7QCDe8qAnVW5e5X8rGvn6m2DCR3JR2yqwVjc/edit](https://docs.google.com/document/d/1Rlqizmk7QCDe8qAnVW5e5X8rGvn6m2DCR3JR2yqwVjc/edit)   
[https://docs.google.com/document/d/11Jqy_GjUGtdXJK94XGsEIK7CP1SnQGdp2eF0wSw9ra8/edit#](https://docs.google.com/document/d/11Jqy_GjUGtdXJK94XGsEIK7CP1SnQGdp2eF0wSw9ra8/edit#)   



