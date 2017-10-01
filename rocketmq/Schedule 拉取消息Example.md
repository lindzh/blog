---
title: RocketMQ 定时拉取消费
date: 2017-06-17 16:20:03
categories:
- rocketmq
tags:
- rocketmq
- 分布式
---

### 定时拉取介绍
如果你采用`DefaultMQPullConsumer` 来消费消息的话，消费消息的queue需要获取，并且消费消息的queue的位点需要业务自己保存，当然也可以利用mq提供的api保存到broker中，另外什么时候来拉取queue中的消息完全由业务自己决定。定时拉取消息的Schedule完全解决了这个问题，业务要处理的就是消费消息，而不用担心位点同步，另外拉取了消息之后什么时候再拉取消息消费完全由业务控制。

#### 利用 `DefaultMQPullConsumer`的步骤

> 第一步：拉取订阅的queue列表

```java
Set<MessageQueue> testTopic = consumer.fetchSubscribeMessageQueues("testTopic");
```

> 第二部：手动从选取的queue中拉取消息，并消费保存queue位点。

#### 利用 定时拉取服务 `MQPullConsumerScheduleService`消费消息

```java
final MQPullConsumerScheduleService scheduleService = new MQPullConsumerScheduleService("GroupName1");

scheduleService.setMessageModel(MessageModel.CLUSTERING);
scheduleService.registerPullTaskCallback("TopicTest1", new PullTaskCallback() {

    @Override
    public void doPullTask(MessageQueue mq, PullTaskContext context) {
        MQPullConsumer consumer = context.getPullConsumer();
        try {

            long offset = consumer.fetchConsumeOffset(mq, false);
            if (offset < 0)
                offset = 0;

            PullResult pullResult = consumer.pull(mq, "*", offset, 32);
            System.out.printf("%s%n", offset + "\t" + mq + "\t" + pullResult);
            switch (pullResult.getPullStatus()) {
                case FOUND:
                    break;
                case NO_MATCHED_MSG:
                    break;
                case NO_NEW_MSG:
                case OFFSET_ILLEGAL:
                    break;
                default:
                    break;
            }
            consumer.updateConsumeOffset(mq, pullResult.getNextBeginOffset());

			//consume message auto
            context.setPullNextDelayTimeMillis(100);
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
});

scheduleService.start();
```

#### `MQPullConsumerScheduleService`与`DefaultMQPullConsumer`的比较
MQPullConsumerScheduleService是DefaultMQPullConsumer的封装，MQPullConsumerScheduleService内部封装了schedule，位点同步，简化了`DefaultMQPullConsumer`的获取queue列表，offset位点保存的额外复杂步骤。
