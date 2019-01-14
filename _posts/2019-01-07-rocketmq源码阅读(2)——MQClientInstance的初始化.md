---
layout: post
title: "rocketmq源码阅读(2)——MQClientInstance的初始化"
date: 2019-01-07
description: "rocketmq源码阅读"
tag: rocketmq
---

## 前言

上篇在读mq生成者相关的代码时。在DefaultMQProducer的初始化最后一步时，代码是**mQClientFactory.start();**。至于MQClientInstance的关键性在上篇文章中也提到了。这篇就从**mQClientFactory.start();**开始吧。先放上相关的代码吧

###字段分析

先来看看MQClientInstance中的重要的变量吧

``` java
//该clientInstance配置，主要包括nameSrv，unitName之类的
private final ClientConfig clientConfig;
//下标，atomicInteger维护
private final int instanceIndex;
//IP@instanceName@UnitName
private final String clientId;
private final long bootTimestamp = System.currentTimeMillis();
//生产者集群缓存
private final ConcurrentMap<String/* group */, MQProducerInner> producerTable = new ConcurrentHashMap<String, MQProducerInner>();
//消费者集群缓存
private final ConcurrentMap<String/* group */, MQConsumerInner> consumerTable = new ConcurrentHashMap<String, MQConsumerInner>();
//顾名思义，MQAdminExtInner的作用是mq后台管理的扩展。通常用在相关的监控中
private final ConcurrentMap<String/* group */, MQAdminExtInner> adminExtTable = new ConcurrentHashMap<String, MQAdminExtInner>();
//netty配置
private final NettyClientConfig nettyClientConfig;
//rpc统一入口
private final MQClientAPIImpl mQClientAPIImpl;
//mq管理入口，包括topic创建接口，offset查询接口
private final MQAdminImpl mQAdminImpl;
//存储topic映射表
private final ConcurrentMap<String/* Topic */, TopicRouteData> topicRouteTable = new ConcurrentHashMap<String, TopicRouteData>();
private final Lock lockNamesrv = new ReentrantLock();
private final Lock lockHeartbeat = new ReentrantLock();
private final ConcurrentMap<String/* Broker Name */, HashMap<Long/* brokerId */, String/* address */>> brokerAddrTable =
    new ConcurrentHashMap<String, HashMap<Long, String>>();
private final ConcurrentMap<String/* Broker Name */, HashMap<String/* address */, Integer>> brokerVersionTable =
    new ConcurrentHashMap<String, HashMap<String, Integer>>();
//单核心线程数的线程池
private final ScheduledExecutorService scheduledExecutorService = Executors.newSingleThreadScheduledExecutor(new ThreadFactory() {
    @Override
    public Thread newThread(Runnable r) {
        return new Thread(r, "MQClientFactoryScheduledThread");
    }
});
//和broker进行rpc通讯时会用到，再讲MQClientAPIImpl会细讲
private final ClientRemotingProcessor clientRemotingProcessor;
//拉取消息服务，用于消费
private final PullMessageService pullMessageService;
//消息消费的负载均衡服务
private final RebalanceService rebalanceService;
//默认producerGroupName为"CLIENT_INNER_PRODUCER"的生产者group
private final DefaultMQProducer defaultMQProducer;
//主要用于消费消息回调时使用，也就是上面的pullMessageService中会用到，之后会细讲
private final ConsumerStatsManager consumerStatsManager;
private final AtomicLong sendHeartbeatTimesTotal = new AtomicLong(0);
private ServiceState serviceState = ServiceState.CREATE_JUST;
private DatagramSocket datagramSocket;
private Random random = new Random();
```



###MQClientInstance启动流程

再来看看**start()**的详细流程：

``` java
public void start() throws MQClientException {
    synchronized (this) {
        switch (this.serviceState) {
            case CREATE_JUST:
                this.serviceState = ServiceState.START_FAILED;
                // If not specified,looking address from name server
                if (null == this.clientConfig.getNamesrvAddr()) {
                    this.mQClientAPIImpl.fetchNameServerAddr();
                }
                // Start request-response channel
                //启动NettyRemotingClient，rocketmq中netty的使用，和namesrv的交互都在此
                this.mQClientAPIImpl.start();
                // Start various schedule tasks
                this.startScheduledTask();
                // Start pull service
                //针对push消息拉取模式,拉取消息服务，主要是从一个阻塞队列中取出pullrequest，遍历consumerTable
                this.pullMessageService.start();
                // Start rebalance service,负载均衡服务
                this.rebalanceService.start();
                // Start push service
                //创建默认producer组CLIENT_INNER_PRODUCER
                this.defaultMQProducer.getDefaultMQProducerImpl().start(false);
                log.info("the client factory [{}] start OK", this.clientId);
                this.serviceState = ServiceState.RUNNING;
                break;
            case RUNNING:
                //第二次来就是Running
                break;
            case SHUTDOWN_ALREADY:
                break;
            case START_FAILED:
                throw new MQClientException("The Factory object[" + this.getClientId() + "] has been created before, and failed.", null);
            default:
                break;
        }
    }
}
```

可以看到主要就这是这几行代码

``` java
this.mQClientAPIImpl.start();
this.startScheduledTask();
this.pullMessageService.start();
this.rebalanceService.start();
```

### MQClientAPI的初始化

MQClientAPI的启动就是remotingClient的启动，也就是RPC的客户端的启动是在这里，相对应的RPC的服务端是在broker端进行初始化。其中涉及到了netty的使用。会在之后一起看看。

### 开启MQClientInstance的相关定时任务

跟踪进去，我们可以发现是这么5个任务

``` java
private void startScheduledTask() {
    //如果未指定namesrvAddress，指定了rocketmq.namesrv.domain.subgroup，会每两分钟去获取nameSrvAddress
    if (null == this.clientConfig.getNamesrvAddr()) {
        this.scheduledExecutorService.scheduleAtFixedRate(new Runnable() {

            @Override
            public void run() {
                try {
                    MQClientInstance.this.mQClientAPIImpl.fetchNameServerAddr();
                } catch (Exception e) {
                    log.error("ScheduledTask fetchNameServerAddr exception", e);
                }
            }
        }, 1000 * 10, 1000 * 60 * 2, TimeUnit.MILLISECONDS);
    }

    //遍历所有的consumerTable和producerTable，更新所有的topic
    this.scheduledExecutorService.scheduleAtFixedRate(new Runnable() {

        @Override
        public void run() {
            try {
                MQClientInstance.this.updateTopicRouteInfoFromNameServer();
            } catch (Exception e) {
                log.error("ScheduledTask updateTopicRouteInfoFromNameServer exception", e);
            }
        }
    }, 10, this.clientConfig.getPollNameServerInterval(), TimeUnit.MILLISECONDS);

    //清理下线broker 发送心跳，维护brokerAddrTable，brokerVersionTable
    this.scheduledExecutorService.scheduleAtFixedRate(new Runnable() {

        @Override
        public void run() {
            try {
                MQClientInstance.this.cleanOfflineBroker();
                MQClientInstance.this.sendHeartbeatToAllBrokerWithLock();
            } catch (Exception e) {
                log.error("ScheduledTask sendHeartbeatToAllBroker exception", e);
            }
        }
    }, 1000, this.clientConfig.getHeartbeatBrokerInterval(), TimeUnit.MILLISECONDS);

    this.scheduledExecutorService.scheduleAtFixedRate(new Runnable() {

        @Override
        public void run() {
            try {
                //持久化消费者集群的queue 的offset
                MQClientInstance.this.persistAllConsumerOffset();
            } catch (Exception e) {
                log.error("ScheduledTask persistAllConsumerOffset exception", e);
            }
        }
    }, 1000 * 10, this.clientConfig.getPersistConsumerOffsetInterval(), TimeUnit.MILLISECONDS);

    this.scheduledExecutorService.scheduleAtFixedRate(new Runnable() {

        @Override
        public void run() {
            try {
                //对于push模式的消费方式，需要动态调整线程池大小
                MQClientInstance.this.adjustThreadPool();
            } catch (Exception e) {
                log.error("ScheduledTask adjustThreadPool exception", e);
            }
        }
    }, 1, 1, TimeUnit.MINUTES);
}
```

总共有个定时任务：

- 获取nameSrv任务

  在没有配置nameSrv的时候，会通过提供的**rocketmq.namesrv.domain**来定时获取nameSrv地址。主要是用于动态调整nameSrv服务器时，不需要修改项目配置

- 更新topic信息任务

  从consumerTable和producerTable中获取topic信息。然后**MQClientInstance.updateTopicRouteInfoFromNameServer()**，这一块在讲producer的时候有写过

- 维护broker信息任务

  主要清理下线broker和添加新增的broker。

- 持久化offset任务

  将本地缓存的offset同步到broker端。保证消息的成功消费。

- 动态调整线程池任务

  根据rebalanceService中所有的ProcessQueue的消息长度，来决定ConsumeMessageService的核心线程数（ProcessQueue存放着需要消费的消息队列，ConsumeMessageService就是消息消费服务，处理消费结果）。不过不知道为什么，在4.3.1的版本中，**incCorePoolSize()**和**decCorePoolSize()**内的方法都被注释掉了。所以这个任务没有起作用。

### 拉取消息启动

也就是pullMessageService。总所周知，rocketmq的消息消费有两种，一个pull模式，一个push模式。pull好理解，根据本地存的offset，去broker上拉取消息。而push模式，是broker端往client端推消息，其实push实际上还是从broker上拉取，正是通过这个**pullMessageService**。

其核心的部分就是

``` java
while (!this.isStopped()) {
    try {
        //从阻塞队列中获取pullRequest
        PullRequest pullRequest = this.pullRequestQueue.take();
        this.pullMessage(pullRequest);
    } catch (InterruptedException ignored) {
    } catch (Exception e) {
        log.error("Pull Message Service Run Method exception", e);
    }
}
```

那么就有问题了？什么时候会往这个阻塞队列中添加PullRequest呢。在看消费者消费流程的时候再来看看

### 负载均衡服务启动

负载均衡涉及到消息拉取相关的服务，内容较多，另外写。

