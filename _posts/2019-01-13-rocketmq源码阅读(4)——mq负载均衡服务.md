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
    if (changed) {
        this.messageQueueChanged(topic, mqSet, allocateResultSet);
    }
}
```

首先是获取broker端所有的messgeQueue。这一部分是直接从本地的缓存中取。在MQClientInstance中会起定时任务，从broker端更新该信息。

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

根据机房进行重新分配:

``` java
//根据机器对MesaageQueue进行分组
//machineRoomResolver为接口，需要具体实现，用于获取相关的machine room唯一标志
Map<String/*machine room */, List<MessageQueue>> mr2Mq = new TreeMap<String, List<MessageQueue>>();
for (MessageQueue mq : mqAll) {
    String brokerMachineRoom = machineRoomResolver.brokerDeployIn(mq);
    if (StringUtils.isNoneEmpty(brokerMachineRoom)) {
        if (mr2Mq.get(brokerMachineRoom) == null) {
            mr2Mq.put(brokerMachineRoom, new ArrayList<MessageQueue>());
        }
        mr2Mq.get(brokerMachineRoom).add(mq);
    } else {
        throw new IllegalArgumentException("Machine room is null for mq " + mq);
    }
}

//根据机器对消费者实例进行分组
Map<String/*machine room */, List<String/*clientId*/>> mr2c = new TreeMap<String, List<String>>();
for (String cid : cidAll) {
    String consumerMachineRoom = machineRoomResolver.consumerDeployIn(cid);
    if (StringUtils.isNoneEmpty(consumerMachineRoom)) {
        if (mr2c.get(consumerMachineRoom) == null) {
            mr2c.put(consumerMachineRoom, new ArrayList<String>());
        }
        mr2c.get(consumerMachineRoom).add(cid);
    } else {
        throw new IllegalArgumentException("Machine room is null for consumer id " + cid);
    }
}

List<MessageQueue> allocateResults = new ArrayList<MessageQueue>();

//1.allocate the mq that deploy in the same machine room with the current consumer
String currentMachineRoom = machineRoomResolver.consumerDeployIn(currentCID);
List<MessageQueue> mqInThisMachineRoom = mr2Mq.remove(currentMachineRoom);
List<String> consumerInThisMachineRoom = mr2c.get(currentMachineRoom);
if (mqInThisMachineRoom != null && !mqInThisMachineRoom.isEmpty()) {
    //具体的allocateMessageQueueStrategy需要传入构造函数中
    allocateResults.addAll(allocateMessageQueueStrategy.allocate(consumerGroup, currentCID, mqInThisMachineRoom, consumerInThisMachineRoom));
}

//2.allocate the rest mq to each machine room if there are no consumer alive in that machine room
for (String machineRoom : mr2Mq.keySet()) {
    if (!mr2c.containsKey(machineRoom)) { // no alive consumer in the corresponding machine room, so all consumers share these queues
        allocateResults.addAll(allocateMessageQueueStrategy.allocate(consumerGroup, currentCID, mr2Mq.get(machineRoom), cidAll));
    }
}

```

根据具体的机房对队列进行分配，具体的意思也注释中也写了，比较注重相关的网络情况。

#### AllocateMessageQueueAveragely

默认的队列分配算法

``` java
//找到该实例对应的下标
int index = cidAll.indexOf(currentCID);
//多余的队列
int mod = mqAll.size() % cidAll.size();

int averageSize =
    mqAll.size() <= cidAll.size() ? 1 : (mod > 0 && index < mod ? mqAll.size() / cidAll.size() + 1 : mqAll.size() / cidAll.size());
//确定下标
int startIndex = (mod > 0 && index < mod) ? index * averageSize : index * averageSize + mod;
//确定范围
int range = Math.min(averageSize, mqAll.size() - startIndex);
for (int i = 0; i < range; i++) {
    result.add(mqAll.get((startIndex + i) % mqAll.size()));
}
```

看下来应该也比较清晰这种分配规则了。举个简单的例子来说。broker集群总共有8个queue：q1,q2,q3,q4,q5,q6,q7,q8。某一个ConsumerGroup有3个实例：c1，c2，c3。那么如果使用这种算法，队列分配结果就是这样的:c1[q1,q2,q3]，c2[q4,q5,q6]，c3[q7,q8]。

#### AllocateMessageQueueAveragelyByCircle

这个算法就是队列进行环形分配（话说看到这个算法就想起大学时碰到的约瑟夫环，不知道你是否还记得，哈哈:-D）

``` java
int index = cidAll.indexOf(currentCID);
for (int i = index; i < mqAll.size(); i++) {
    if (i % cidAll.size() == index) {
        result.add(mqAll.get(i));
    }
}
```

代码言简意赅

#### AllocateMessageQueueByConfig

通过配置来分配队列

``` java
private List<MessageQueue> messageQueueList;

@Override
public List<MessageQueue> allocate(String consumerGroup, String currentCID, List<MessageQueue> mqAll, List<String> cidAll) {
    //自定义
    return this.messageQueueList;
}
```

不清楚这种算法的适用场景。感觉也就本机测试的时候可以用用。

#### AllocateMessageQueueByMachineRoom

``` java
for (MessageQueue mq : mqAll) {
    String[] temp = mq.getBrokerName().split("@");
    //统计所有broker有效的messageQueue，consumeridcs为初始化时配置
    if (temp.length == 2 && consumeridcs.contains(temp[0])) {
        premqAll.add(mq);
    }
}

int mod = premqAll.size() / cidAll.size();
int rem = premqAll.size() % cidAll.size();
int startIndex = mod * currentIndex;
int endIndex = startIndex + mod;
for (int i = startIndex; i < endIndex; i++) {
    result.add(mqAll.get(i));
}
if (rem > currentIndex) {
    result.add(premqAll.get(currentIndex + mod * cidAll.size()));
}
```

其实代码不难看懂，但真不清楚这种分配策略适合什么样的场景。

#### AllocateMessageQueueConsistentHash

根据一致性hash算法来分配队列。虽然感觉在rocketmq中，这种算法的意义不大，因为负载均衡服务默认20s执行一次，感觉一致性hash算法的适用场景是那种，集群如果其中单点挂了会出现集群雪崩这种严重的问题。不过我们可以看看其对一致性hash算法的实现，也正好复习一下一致性hash算法。

``` java
//org.apache.rocketmq.client.consumer.rebalance.AllocateMessageQueueConsistentHash
//虚拟节点个数，默认为10
private final int virtualNodeCnt;
//hash算法实现类，默认为org.apache.rocketmq.common.consistenthash.ConsistentHashRouter.MD5Hash
private final HashFunction customHashFunction;
```

``` java
Collection<ClientNode> cidNodes = new ArrayList<ClientNode>();
for (String cid : cidAll) {
    cidNodes.add(new ClientNode(cid));
}

//构建hash环
final ConsistentHashRouter<ClientNode> router; 
if (customHashFunction != null) {
    router = new ConsistentHashRouter<ClientNode>(cidNodes, virtualNodeCnt, customHashFunction);
} else {
    router = new ConsistentHashRouter<ClientNode>(cidNodes, virtualNodeCnt);
}

List<MessageQueue> results = new ArrayList<MessageQueue>();
//对每个messageQueue计算对应的Client节点
for (MessageQueue mq : mqAll) {
    ClientNode clientNode = router.routeNode(mq.toString());
    if (clientNode != null && currentCID.equals(clientNode.getKey())) {
        results.add(mq);
    }
}
```

主要看看**hash环的构建**以及**router.routeNode()**就行

hash环的构建其实就是将cidNode通过hash加入到hash环中。

``` java
//org.apache.rocketmq.common.consistenthash.ConsistentHashRouter#addNode
//统计pNode(可以理解为cid，也就是实例名称)目前含有的虚拟节点的个数，用于创建虚拟节点时传入的下标
int existingReplicas = getExistingReplicas(pNode);
for (int i = 0; i < vNodeCount; i++) {
    VirtualNode<T> vNode = new VirtualNode<T>(pNode, i + existingReplicas);
    //ring为一个TreeMap
    ring.put(hashFunction.hash(vNode.getKey()), vNode);
}
```

router.routerNode():

``` java
//找到大于或等于传入的MessageQueue的hash值的第一个虚拟节点。类似于C++中的lower_bound()
Long hashVal = hashFunction.hash(objectKey);
SortedMap<Long, VirtualNode<T>> tailMap = ring.tailMap(hashVal);
Long nodeHashVal = !tailMap.isEmpty() ? tailMap.firstKey() : ring.firstKey();
return ring.get(nodeHashVal).getPhysicalNode();
```



当然，队列分配算法也没有绝对的好坏之差，这也只是官方提供参考的几种，我们自己也可以实现。还是得根据实际的业务场景进行合理的选择。毕竟，适合的才是最好的嘛：）

### ProcessQueue的更新

在分配好queue后，还需要更新ProcessQueue信息，才能开始消费消息，**ProcessQueue**和**MessageQueue**是一一对应的，它存储的是这个队列的消费快照，包括offset，拉取的消息实体等。在**MQClientInstance**中，提到了**PullMessageService**，其中有一个存放pullRequest的阻塞队列，其中第一次分发pullRequest也就是在这边执行的。来看一下核心的代码

``` java
//org/apache/rocketmq/client/impl/consumer/RebalanceImpl#updateProcessQueueTableInRebalance
List<PullRequest> pullRequestList = new ArrayList<PullRequest>();
//进行添加，遍历分配的queue，以及pullrequest的添加
for (MessageQueue mq : mqSet) {
    //如果当前没有该MessageQueue
    if (!this.processQueueTable.containsKey(mq)) {
        //对于顺序消费的情况
        if (isOrder && !this.lock(mq)) {
            log.warn("doRebalance, {}, add a new mq failed, {}, because lock failed", consumerGroup, mq);
            continue;
        }

        //移除失效的queue对应的offset
        this.removeDirtyOffset(mq);
        ProcessQueue pq = new ProcessQueue();
        //计算消费下标，有三种策略，CONSUME_FROM_LAST_OFFSET，CONSUME_FROM_FIRST_OFFSET，CONSUME_FROM_TIMESTAMP
        long nextOffset = this.computePullFromWhere(mq);
        if (nextOffset >= 0) {
            ProcessQueue pre = this.processQueueTable.putIfAbsent(mq, pq);
            if (pre != null) {
                log.info("doRebalance, {}, mq already exists, {}", consumerGroup, mq);
            } else {
                log.info("doRebalance, {}, add a new mq, {}", consumerGroup, mq);
                PullRequest pullRequest = new PullRequest();
                pullRequest.setConsumerGroup(consumerGroup);
                pullRequest.setNextOffset(nextOffset);
                pullRequest.setMessageQueue(mq);
                pullRequest.setProcessQueue(pq);
                pullRequestList.add(pullRequest);
                changed = true;
            }
        } else {
            log.warn("doRebalance, {}, add new mq failed, {}", consumerGroup, mq);
        }
    }
}

//分发pullRequest
this.dispatchPullRequest(pullRequestList);
```

代码也比较直白，初始化的过程到这里也差不多了。接下来就是看看PullMessageService是怎么处理PullRequest的。

### 思考

1.最近一直在想负载均衡服务为什么要放在客户端做，而不放在服务端做？rocketmq的设计是让客户端拉取所有的消费者组信息，然后根据相应的算法做负载均衡。如果让服务端来做分配算法，再分发给所有的消费者，感觉会更合适？其实不然，其实上面我们也提到，会有多种分配算法，如果我们要修改策略，只需要让客户端重启即可。如果负载均衡放在broker，那可就得重启broker了！！！