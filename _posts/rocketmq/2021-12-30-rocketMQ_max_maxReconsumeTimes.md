---
title: RocketMQ 最大消费次数maxReconsumeTimes
tagline: ""
category : rocketmq
layout: post
tags : [rocketmq]
---

默认事务消息最大次数transactionCheckMax=15，以及间隔时间transactionCheckInterval=60*1000
```
    /**
     * The maximum number of times the message was checked, if exceed this value, this message will be discarded.
     */
    @ImportantField
    private int transactionCheckMax = 15;

    /**
     * Transaction message check interval.
     */
    @ImportantField
    private long transactionCheckInterval = 60 * 1000;
```

### 最大消费次数

####  首先看下普通消息与顺序消息有何不同
```
consumer.start()-->this.defaultMQPushConsumerImpl.start()

```
不同的消费这对应不同的Service: ConsumeMessageOrderlyService和ConsumeMessageConcurrentlyService
```
//
if (this.getMessageListenerInner() instanceof MessageListenerOrderly) {
    this.consumeOrderly = true;
    this.consumeMessageService =
        new ConsumeMessageOrderlyService(this, (MessageListenerOrderly) this.getMessageListenerInner());
} else if (this.getMessageListenerInner() instanceof MessageListenerConcurrently) {
    this.consumeOrderly = false;
    this.consumeMessageService =
        new ConsumeMessageConcurrentlyService(this, (MessageListenerConcurrently) this.getMessageListenerInner());
}
this.consumeMessageService.start()
```
### 结论：
1,不论是顺序消息还是普通消息，最大消费次数都是maxReconsumeTimes   
2，二者使用的getMaxReconsumeTimes()方法必然不同,   
3，ConsumeMessageConcurrentlyService使用的是DefaultMQPushConsumerImpl的getMaxReconsumeTimes()   
4，ConsumeMessageOrderlyService 使用的是自己内部的getMaxReconsumeTimes()   
5, 从3，4看起来是挺奇怪的一个设计，按理这个既然是两种不同的实现，为何不都放ConsumeMessageService，然后各自实现自己的getMaxReconsumeTimes()呢   
6，默认情况下maxReconsumeTimes都是-1, 但是普通消息其实maxReconsumeTimes=16; 而且顺序消息maxReconsumeTimes= Integer.MAX_VALUE，也就是无限次   
7,如果达到maxReconsumeTimes次消息后，就真的不能再消费到了吗？其实还有DLQ


#### 看下两个getMaxReconsumeTimes()的实现  
首先是ConsumeMessageOrderlyService，这里很明显，如果不配置默认值是-1，也就意味着如果消费失败，会无限次消费重试
```
  private int getMaxReconsumeTimes() {
      // default reconsume times: Integer.MAX_VALUE
      if (this.defaultMQPushConsumer.getMaxReconsumeTimes() == -1) {
          return Integer.MAX_VALUE;
      } else {
          return this.defaultMQPushConsumer.getMaxReconsumeTimes();
      }
  }
```

ConsumeMessageConcurrentlyService的在DefaultMQPushConsumerImpl里实现
```
  private int getMaxReconsumeTimes() {
      // default reconsume times: 16
      if (this.defaultMQPushConsumer.getMaxReconsumeTimes() == -1) {
          return 16;
      } else {
          return this.defaultMQPushConsumer.getMaxReconsumeTimes();
      }
  }
```



感觉涉及挺奇怪的 虽然这两service 构造函数都需要DefaultMQPushConsumerImpl作为参数,但是
