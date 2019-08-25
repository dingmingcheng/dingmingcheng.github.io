---
layout: post
title: "rocketmq源码阅读(3)——mq消费者初始化"
date: 2019-01-13
description: "rocketmq源码阅读"
tag: rocketmq
---

## 写在前面

最近开始看了一些rocketmq的源码，在正式讲消费者初始化过程之前先写写自己的感受吧。虽说已经写了两篇了，但阅读以及整理的时候，还是感觉有很多问题。之前看源码更多的是去了解相关的流程。在写文章时，还是能体会到自己太弱，无法理解相关设计的精妙之处，仅仅是能看明白流程。说到精妙之处，比如说其中设计模式的应用，相关锁的使用原因，带着问题去看才会发现不懂的还有很多。另外，在整理的时候，感觉似乎是太抠一些不太重要的细节了，个人感觉效率太低，没有太大的必要。不可置否，像rocketmq这么精妙的中间件，肯定无法很快参透，不过通过写博客确实能推着自己去学习，思考，因为要把一个东西描述出来可比观察一个东西来着复杂。现在还只是停在rocketmq-client的部分，在深入broker端的时候，肯定会有更多的问题。如有发现不当之处，但凡指出。

rocketmq的消费者分为两种，push模式和pull模式，但在大部分的工作场景中，都是使用的push模式。而且，push和pull并没有太大的区别，其实两种方式都是client端到broker端拉取消息，只不过push模式的拉取请求是程序触发的。所以消费者部分，主要是写写push模式。

之前已经看过了Producer的初始化以及MQClientInstance部分，其实Consumer也是大同小异

### 整体启动流程

``` java
this.checkConfig();
//复制topic的信息到负载均衡服务中
this.copySubscription();

if (this.defaultMQPushConsumer.getMessageModel() == MessageModel.CLUSTERING) {
    this.defaultMQPushConsumer.changeInstanceNameToPID();
}

//...
this.mQClientFactory = MQClientManager.getInstance().getAndCreateMQClientInstance(this.defaultMQPushConsumer, this.rpcHook);

//配置负载均衡服务
this.rebalanceImpl.setConsumerGroup(this.defaultMQPushConsumer.getConsumerGroup());
this.rebalanceImpl.setMessageModel(this.defaultMQPushConsumer.getMessageModel());
this.rebalanceImpl.setAllocateMessageQueueStrategy(this.defaultMQPushConsumer.getAllocateMessageQueueStrategy());
this.rebalanceImpl.setmQClientFactory(this.mQClientFactory);

//拉消息包装
this.pullAPIWrapper = new PullAPIWrapper(
    mQClientFactory,
    this.defaultMQPushConsumer.getConsumerGroup(), isUnitMode());
this.pullAPIWrapper.registerFilterMessageHook(filterMessageHookList);

if (this.defaultMQPushConsumer.getOffsetStore() != null) {
    this.offsetStore = this.defaultMQPushConsumer.getOffsetStore();
} else {
    switch (this.defaultMQPushConsumer.getMessageModel()) {
        case BROADCASTING:
            //offset存储在本地LocalFileOffsetStore.LOCAL_OFFSET_STORE_DIR
            this.offsetStore = new LocalFileOffsetStore(this.mQClientFactory, this.defaultMQPushConsumer.getConsumerGroup());
            break;
        case CLUSTERING:
            this.offsetStore = new RemoteBrokerOffsetStore(this.mQClientFactory, this.defaultMQPushConsumer.getConsumerGroup());
            break;
        default:
            break;
    }
    this.defaultMQPushConsumer.setOffsetStore(this.offsetStore);
}
this.offsetStore.load();

if (this.getMessageListenerInner() instanceof MessageListenerOrderly) {
    //顺序消费
    this.consumeOrderly = true;
    this.consumeMessageService =
        new ConsumeMessageOrderlyService(this, (MessageListenerOrderly) this.getMessageListenerInner());
} else if (this.getMessageListenerInner() instanceof MessageListenerConcurrently) {
    //无序消费
    this.consumeOrderly = false;
    this.consumeMessageService =
        new ConsumeMessageConcurrentlyService(this, (MessageListenerConcurrently) this.getMessageListenerInner());
}

//消费服务开启
this.consumeMessageService.start();

boolean registerOK = mQClientFactory.registerConsumer(this.defaultMQPushConsumer.getConsumerGroup(), this);
if (!registerOK) {
    this.serviceState = ServiceState.CREATE_JUST;
    this.consumeMessageService.shutdown();
    throw new MQClientException("The consumer group[" + this.defaultMQPushConsumer.getConsumerGroup()
                                + "] has been created before, specify another name please." + FAQUrl.suggestTodo(FAQUrl.GROUP_NAME_DUPLICATE_URL),
                                null);
}

mQClientFactory.start();

this.updateTopicSubscribeInfoWhenSubscriptionChanged();
this.mQClientFactory.checkClientInBroker();
//向broker发送心跳
this.mQClientFactory.sendHeartbeatToAllBrokerWithLock();
//可以运行rebalance服务
this.mQClientFactory.rebalanceImmediately();
```

其实从执行顺序上来看，主要有这几个部分。

1. 初始化MQClientInstance实例
2. 初始化客户端负载均衡服务，即rebalanceImpl
3. 初始化pullAPIWrapper，pullAPIWrapper用于处理拉取下来的消息
4. offset更新策略选择
5. 初始化以及启动消费服务，即consumeMessageService，用于进行消息的消费。
6. MQClientInstance的初始化

接下来一部分一部分来看吧。其中第1部分和第6部分在生产者初始化中提过，就不赘述了。

### rebalance服务初始化

rebalance服务是干什么的？首先得明确，rebalance服务只是用于消费者的。众所周知，在broker端，一个topic会对应多个queue，一个topic也会对应多个消费者(即ConsumerGroup)，而单个ConsumerGroup一般也会对应多个consumer(即应用的多节点部署)。那么每一个节点的应用怎么知道“我”该消费哪一个queue的消息呢？rebalance服务其实就是解决这个问题的。

rebalance也根据pull和push，划分为两个类—RebalancePullImpl和RebalancePushImpl。

本文就先简单介绍一下，之后会拆出来细看

### pullAPIWrapper初始化

pullAPIWrapper又是干什么的呢？自此也简单讲讲消费的流程。在MQClientInstance中我们看到了pullMessageService。pullMessageService会调用**DefaultMQPushConsumerImpl.pullMessage()**函数。这里的pullMessage其实是传入了一个回调接口**PullCallback**，rpc调用返回的结果**PullResult**即交由pullAPIWrapper处理，再把具体的消息内容交由ConsumerMessageService处理。

### offset更新策略选择

在所有的mq中，offset都是很重要的。我们也可以看到，广播消费方式和集群消费模式(默认)使用的offset更新策略是不一样的。广播模式中的offset都是存放在本地的文件中，而集群模式是在broker端进行持久化存储的。

### consumeMessageService初始化

rocketmq中可以配置两种消息消费的方式，分为顺序消费和非顺序消费。对应这两种方式，也有两个不同的实现类。**ConsumeMessageOrderlyService**和**ConsumeMessageOrderlyService**。

## 思考

因为本文只涉及到初始化部分，和Producer类似，所以主要讲了讲相关的概念以及个人的理解，为后面的内容做好铺垫。之后会针对消费分为两块来写，**客户端的负载均衡**以及**消息消费的流程**。