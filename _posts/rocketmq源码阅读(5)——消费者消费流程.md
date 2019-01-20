---
layout: post
title: "rocketmq源码阅读(5)——消费者消费流程"
date: 2019-01-20
description: "rocketmq源码阅读"
tag: rocketmq
---

## 前言

在消费者初始化过程中，我们看到了**PullRequest**的分发，接下来就是看看具体的消费流程。大体的流程是这样的：

1. PullRequest的处理
2. 回调PullCallBack处理pullResult
3. ConsumeMessageService处理消息

### PullRequest的处理

PullMessageService在拿到PullRequest后，直接转发给DefauleMQPushConsumerImpl进行处理：

在拿到PullRequest后，DefauleMQPushConsumerImpl不会马上就去进行RPC调用。如果ProcessQueue中缓存的条数大于1000条，或者大于100M，会停50微秒再将PullRequest重新投递到PullMessageService中，以达到流量控制，避免压力过大。当然，以上数值均为默认值。

这里主要就是一些细节的处理，参数的拼凑，再交由pullAPIWrapper进行RPC通信，当然会传入回调接口。细节的代码就不贴了。

在PullRequest的中，还加杂了PullCallBack的初始化，这个会在看处理pullRequest时再看看。

### 回调PullCallBack处理PullResult

首先来看看PullCallBack的匿名内部类的实现吧

``` java
PullCallback pullCallback = new PullCallback() {
    @Override
    public void onSuccess(PullResult pullResult) {
        if (pullResult != null) {
            pullResult = DefaultMQPushConsumerImpl.this.pullAPIWrapper.processPullResult(pullRequest.getMessageQueue(), pullResult, subscriptionData);

            switch (pullResult.getPullStatus()) {
                case FOUND:
                    long prevRequestOffset = pullRequest.getNextOffset();
                    pullRequest.setNextOffset(pullResult.getNextBeginOffset());
                    long pullRT = System.currentTimeMillis() - beginTimestamp;
                    DefaultMQPushConsumerImpl.this.getConsumerStatsManager().incPullRT(pullRequest.getConsumerGroup(),
                                                                                       pullRequest.getMessageQueue().getTopic(), pullRT);

                    long firstMsgOffset = Long.MAX_VALUE;
                    if (pullResult.getMsgFoundList() == null || pullResult.getMsgFoundList().isEmpty()) {
                        DefaultMQPushConsumerImpl.this.executePullRequestImmediately(pullRequest);
                    } else {
                        //第一条消息的offset
                        firstMsgOffset = pullResult.getMsgFoundList().get(0).getQueueOffset();

                        DefaultMQPushConsumerImpl.this.getConsumerStatsManager().incPullTPS(pullRequest.getConsumerGroup(),
                                                                                            pullRequest.getMessageQueue().getTopic(), pullResult.getMsgFoundList().size());

                        //拉取来的消息放入processQueue中
                        boolean dispatchToConsume = processQueue.putMessage(pullResult.getMsgFoundList());
                        //提交给consumerMessageService处理
                        DefaultMQPushConsumerImpl.this.consumeMessageService.submitConsumeRequest(
                            pullResult.getMsgFoundList(),
                            processQueue,
                            pullRequest.getMessageQueue(),
                            dispatchToConsume);

                        if (DefaultMQPushConsumerImpl.this.defaultMQPushConsumer.getPullInterval() > 0) {
                            DefaultMQPushConsumerImpl.this.executePullRequestLater(pullRequest,
                                                                                   DefaultMQPushConsumerImpl.this.defaultMQPushConsumer.getPullInterval());
                        } else {
                            DefaultMQPushConsumerImpl.this.executePullRequestImmediately(pullRequest);
                        }
                    }

                    if (pullResult.getNextBeginOffset() < prevRequestOffset
                        || firstMsgOffset < prevRequestOffset) {
                        log.warn(
                            "[BUG] pull message result maybe data wrong, nextBeginOffset: {} firstMsgOffset: {} prevRequestOffset: {}",
                            pullResult.getNextBeginOffset(),
                            firstMsgOffset,
                            prevRequestOffset);
                    }

                    break;
                case NO_NEW_MSG:
                    pullRequest.setNextOffset(pullResult.getNextBeginOffset());

                    DefaultMQPushConsumerImpl.this.correctTagsOffset(pullRequest);

                    DefaultMQPushConsumerImpl.this.executePullRequestImmediately(pullRequest);
                    break;
                case NO_MATCHED_MSG:
                    pullRequest.setNextOffset(pullResult.getNextBeginOffset());

                    DefaultMQPushConsumerImpl.this.correctTagsOffset(pullRequest);

                    DefaultMQPushConsumerImpl.this.executePullRequestImmediately(pullRequest);
                    break;
                case OFFSET_ILLEGAL:
                    log.warn("the pull request offset illegal, {} {}",
                             pullRequest.toString(), pullResult.toString());
                    pullRequest.setNextOffset(pullResult.getNextBeginOffset());

                    pullRequest.getProcessQueue().setDropped(true);
                    DefaultMQPushConsumerImpl.this.executeTaskLater(new Runnable() {

                        @Override
                        public void run() {
                            try {
                                DefaultMQPushConsumerImpl.this.offsetStore.updateOffset(pullRequest.getMessageQueue(), pullRequest.getNextOffset(), false);

                                DefaultMQPushConsumerImpl.this.offsetStore.persist(pullRequest.getMessageQueue());

                                DefaultMQPushConsumerImpl.this.rebalanceImpl.removeProcessQueue(pullRequest.getMessageQueue());

                                log.warn("fix the pull request offset, {}", pullRequest);
                            } catch (Throwable e) {
                                log.error("executeTaskLater Exception", e);
                            }
                        }
                    }, 10000);
                    break;
                default:
                    break;
            }
        }
    }

    @Override
    public void onException(Throwable e) {
        if (!pullRequest.getMessageQueue().getTopic().startsWith(MixAll.RETRY_GROUP_TOPIC_PREFIX)) {
            log.warn("execute the pull request exception", e);
        }

        DefaultMQPushConsumerImpl.this.executePullRequestLater(pullRequest, PULL_TIME_DELAY_MILLS_WHEN_EXCEPTION);
    }
};

```

可以看到总共会返回4中拉取状态:FOUND,NO_NEW_MSG,NO_MATCHED_MSG,OFFSET_ILLEGAL。

对于中间两种，会更新本地的offset，然后重新投递PullRequest。对于OFFSET_ILLEGAL，会清理该ProcessQueue。

主要还是来看看FOUND的情况

