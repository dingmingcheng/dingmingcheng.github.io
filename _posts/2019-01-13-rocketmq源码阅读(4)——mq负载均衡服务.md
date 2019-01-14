---
layout: post
title: "rocketmq源码阅读(4)——mq负载均衡服务"
date: 2019-01-13
description: "rocketmq源码阅读"
tag: rocketmq
---

## 前言

上一文在消费者初始化中，提到了客户端的负载均衡服务。再简单介绍一下负载均衡服务

rebalance服务主要用于每个消费者实例匹配一个或者多个queue，以保证broker端的queue只对应一个实例，避免多个实例消费相同的queue。

rebalance主要包括**RebalancePullImpl**和**RebalancePushImpl**，位于**org.apache.rocketmq.client.impl.consumer**下，还有就是队列分配算法，在**org.apache.rocketmq.client.consumer.rebalance**下。主要包含了6中不同的分配算法。下面逐个来写吧。（本文主要分析push模式下的负载均衡实现）

### 相关参数

``` java
//key代表当前实例对应处理的队列，value代表处理队列快照。代表该应用所对应的queue
protected final ConcurrentMap<MessageQueue, ProcessQueue> processQueueTable = new ConcurrentHashMap<MessageQueue, ProcessQueue>(64);
//topic与对应的所有queue的存储
protected final ConcurrentMap<String/* topic */, Set<MessageQueue>> topicSubscribeInfoTable =
    new ConcurrentHashMap<String, Set<MessageQueue>>();
//topic对应的相关信息，初始化DefaultMQPushConsumer时copy进去
protected final ConcurrentMap<String /* topic */, SubscriptionData> subscriptionInner =
    new ConcurrentHashMap<String, SubscriptionData>();
protected String consumerGroup;
protected MessageModel messageModel;
//队列分配算法
protected AllocateMessageQueueStrategy allocateMessageQueueStrategy;
protected MQClientInstance mQClientFactory;
```

###启动流程

MQClientInstance在start()时，会调用**rebalanceService.start();**。跟踪下去，会调用**MQClientInstance.doRebalance()**

``` java
public void doRebalance() {
    for (Map.Entry<String, MQConsumerInner> entry : this.consumerTable.entrySet()) {
        //针对每个消费者group执行负载均衡服务
        MQConsumerInner impl = entry.getValue();
        if (impl != null) {
            try {
                impl.doRebalance();
            } catch (Throwable e) {
                log.error("doRebalance exception", e);
            }
        }
    }
}
```

接着往下走

``` java
//做负载均衡，判断是否是顺序消费
public void doRebalance(final boolean isOrder) {
    //获取该消费者group下的topic信息
    Map<String, SubscriptionData> subTable = this.getSubscriptionInner();
    if (subTable != null) {
        for (final Map.Entry<String, SubscriptionData> entry : subTable.entrySet()) {
            final String topic = entry.getKey();
            try {
                //对每一个topic进行负载均衡
                this.rebalanceByTopic(topic, isOrder);
            } catch (Throwable e) {
                //如果不是retry队列
                if (!topic.startsWith(MixAll.RETRY_GROUP_TOPIC_PREFIX)) {
                    log.warn("rebalanceByTopic Exception", e);
                }
            }
        }
    }
    this.truncateMessageQueueNotMyTopic();
}
```

### 负载均衡实现

接着往下走，就是队列分配的核心实现。我们来看一下集群模式的负载均衡实现(去除了日志)

``` java
//所有的messageQueue
Set<MessageQueue> mqSet = this.topicSubscribeInfoTable.get(topic);
//所有的实例名称 ip@pid
List<String> cidAll = this.mQClientFactory.findConsumerIdList(topic, consumerGroup);

if (mqSet != null && cidAll != null) {
    List<MessageQueue> mqAll = new ArrayList<MessageQueue>();
    mqAll.addAll(mqSet);

    Collections.sort(mqAll);
    Collections.sort(cidAll);

    //默认为AllocateMessageQueueAveragely，在DefaultMQPushConsumer构造函数中生成
    AllocateMessageQueueStrategy strategy = this.allocateMessageQueueStrategy;

    List<MessageQueue> allocateResult = null;
    try {
        allocateResult = strategy.allocate(
            this.consumerGroup,
            this.mQClientFactory.getClientId(),
            mqAll,
            cidAll);
    } catch (Throwable e) {
        return;
    }

    Set<MessageQueue> allocateResultSet = new HashSet<MessageQueue>();
    if (allocateResult != null) {
        allocateResultSet.addAll(allocateResult);
    }

    //更新处理队列
    boolean changed = this.updateProcessQueueTableInRebalance(topic, allocateResultSet, isOrder);
    if (changed) 「
        this.messageQueueChanged(topic, mqSet, allocateResultSet);
    }
}
```

首先是获取broker端所有的messgeQueue。这一部分是直接从本地的缓存中取。在MQClientInstance中会定时从broker端更新该信息。

另外就是获取所有在broker上注册了该topic的实例名称。即MQClientInstance的唯一标志：IP@PID(默认)

接着会这两个List进行排序，用于后续的队列分配算法。

最后会通过**this.updateProcessQueueTableInRebalance**来更新ProcessQueue。

接着主要来仔细看看队列分配算法，以及ProcessQueue的更新逻辑

### 队列分配算法

在初始化DefaultMQPushConsumer时可以进行设置队列分配算法。官方给我们提供了6种算法：

- AllocateMachineRoomNearby
- AllocateMessageQueueAveragely（默认）
- AllocateMessageQueueAveragelyByCircle
- AllocateMessageQueueByConfig
- AllocateMessageQueueByMachineRoom
- AllocateMessageQueueConsistentHash

#### AllocateMachineRoomNearby

#### AllocateMessageQueueAveragely

#### AllocateMessageQueueAveragelyByCircle

#### AllocateMessageQueueByConfig

#### AllocateMessageQueueByMachineRoom

#### AllocateMessageQueueConsistentHash



