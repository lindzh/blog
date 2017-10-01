---
title: RocketMQ消息发送解析
date: 2017-05-08 16:20:03
categories:
- rocketmq
tags:
- rocketmq
- 分布式
---

Rocketmq的消息发送涉及三个角色，nameserver，broker和producer。nameserver：保存topic queue路由信息，broker：存储topic的queue消息，producer：也无妨发送消息client。发送消息的过程如下：

1. client从nameserver获取topic queue路由列表信息
2. client选择发送的queue
3. client发送消息到queue（Broker中）
![](images/msg_send.png)

#### client获取topic路由信息
client发送消息的时候会从本地的cache中拿topic的路由信息，如果路由信息没在本地，client会从nameserver获取路由信息，获取的请求和返回如下，获取完毕后，该topic的路由信息会保存在本地。

先从本地缓存获取，获取不到再从nameserver获取，并加入本地cache

```java
TopicPublishInfo topicPublishInfo = this.topicPublishInfoTable.get(topic);
if (null == topicPublishInfo || !topicPublishInfo.ok()) {
    this.topicPublishInfoTable.putIfAbsent(topic, new TopicPublishInfo());
    this.mQClientFactory.updateTopicRouteInfoFromNameServer(topic);
    topicPublishInfo = this.topicPublishInfoTable.get(topic);
}
```

从nameserver获取路由信息

```java
topicRouteData = this.mQClientAPIImpl.getTopicRouteInfoFromNameServer(topic, 1000 * 3);
boolean changed = topicRouteDataIsChange(old, topicRouteData);
// Update Pub info
{
    TopicPublishInfo publishInfo = topicRouteData2TopicPublishInfo(topic, topicRouteData);
    publishInfo.setHaveTopicRouterInfo(true);
    Iterator<Entry<String, MQProducerInner>> it = this.producerTable.entrySet().iterator();
    while (it.hasNext()) {
        Entry<String, MQProducerInner> entry = it.next();
        MQProducerInner impl = entry.getValue();
        if (impl != null) {
            impl.updateTopicPublishInfo(topic, publishInfo);
        }
    }
}

// Update sub info
{
    Set<MessageQueue> subscribeInfo = topicRouteData2TopicSubscribeInfo(topic, topicRouteData);
    Iterator<Entry<String, MQConsumerInner>> it = this.consumerTable.entrySet().iterator();
    while (it.hasNext()) {
        Entry<String, MQConsumerInner> entry = it.next();
        MQConsumerInner impl = entry.getValue();
        if (impl != null) {
            impl.updateTopicSubscribeInfo(topic, subscribeInfo);
        }
    }
}
log.info("topicRouteTable.put TopicRouteData[{}]", cloneTopicRouteData);
//放入缓存中
this.topicRouteTable.put(topic, cloneTopicRouteData);

```
路由信息格式

```java
private String orderTopicConf;
private List<QueueData> queueDatas;
private List<BrokerData> brokerDatas;
private HashMap<String/* brokerAddr */, List<String>/* Filter Server */> filterServerTable;

//QueueData
private String brokerName;
private int readQueueNums;
private int writeQueueNums;
private int perm;
private int topicSynFlag;

//BrokerData
private String cluster;
private String brokerName;
private HashMap<Long/* brokerId */, String/* broker address */> brokerAddrs;

```
orderTopicConf格式：

brokername:writequeueNumbers;brokername:writequeueNumbers

说明：orderTopicConf表示的是整个topic的queue排列顺序，broker宕机并不会删除这个conf

#### client 选择queue

##### 路由信息与orderconf转换为queue列表

```java
//优先根据order topic conf
if (route.getOrderTopicConf() != null && route.getOrderTopicConf().length() > 0) {
    String[] brokers = route.getOrderTopicConf().split(";");
    for (String broker : brokers) {
        String[] item = broker.split(":");
        int nums = Integer.parseInt(item[1]);
        for (int i = 0; i < nums; i++) {
            MessageQueue mq = new MessageQueue(topic, item[0], i);
            info.getMessageQueueList().add(mq);
        }
    }

    info.setOrderTopic(true);
} else {
    List<QueueData> qds = route.getQueueDatas();
    Collections.sort(qds);
    for (QueueData qd : qds) {
        if (PermName.isWriteable(qd.getPerm())) {
            BrokerData brokerData = null;
            for (BrokerData bd : route.getBrokerDatas()) {
                if (bd.getBrokerName().equals(qd.getBrokerName())) {
                    brokerData = bd;
                    break;
                }
            }

            if (null == brokerData) {
                continue;
            }

            if (!brokerData.getBrokerAddrs().containsKey(MixAll.MASTER_ID)) {
                continue;
            }

            for (int i = 0; i < qd.getWriteQueueNums(); i++) {
                MessageQueue mq = new MessageQueue(topic, qd.getBrokerName(), i);
                info.getMessageQueueList().add(mq);
            }
        }
    }

    info.setOrderTopic(false);
}

```

#####  一般消息（无序)选择topic

```java
public MessageQueue selectOneMessageQueue() {
    int index = this.sendWhichQueue.getAndIncrement();
    int pos = Math.abs(index) % this.messageQueueList.size();
    if (pos < 0)
        pos = 0;
    return this.messageQueueList.get(pos);
}

```
无序消息的一般选择queue如上所示，都是采用模的办法，但是rocketmq有部分的优化，避免单个broker的热点问题。具体可以阅读相关代码。

##### 顺序消息 选择topic
可以自己选择一个queue传入send方法，或者实现MessageQueueSelector的方式选择queue。常见该需求场景：局部要求顺序，如订单支付，订单发货等。

```java
public interface MessageQueueSelector {
    MessageQueue select(final List<MessageQueue> mqs, final Message msg, final Object arg);
}
```


#### client发送消息到queue
rocketmq提供的api

```java
//简单无序发送
SendResult send(final Message msg) throws MQClientException, RemotingException, MQBrokerException,
	InterruptedException;
	
//有序发送，自己选择queue，自定义MessageQueueSelector
SendResult send(final Message msg, final MessageQueueSelector selector, final Object arg)
	throws MQClientException, RemotingException, MQBrokerException, InterruptedException;

//指定queue发送支持
List<MessageQueue> fetchPublishMessageQueues(final String topic) throws MQClientException;

//指定queue发送
SendResult send(final Message msg, final MessageQueue mq) throws MQClientException,
    RemotingException, MQBrokerException, InterruptedException;
```
其他异步，指定超时时间支持，请参考client提供的api

#### 应对broker宕机
如果broker宕机，client能较快感知到broker宕机的是发送到该broker的消息会失败。如果没有消息发送，则需要通过nameserver的自动清理broker的心跳更新定时任务更新topic路由信息，client定时任务拉取到才能感知到。时间预测：nameserver清理broker时间2分钟执行一次，client定时拉取时间默认30s，可配置，所以一般需要接近3分钟client才能感知到。

无序消息应对宕机:如果设置了重试次数，消息会重新发送到其他broker。默认重试次数：3

#### 新的broker加入
broker加入到集群或者在broker上面创建了topic，broker会同步该broker上所有的topic信息到nameserver，这也是broker和nameserver的心跳。client发送消息首先会从本地缓存中拿到topic的路由信息，如果本地没有，会从nameserver通过传入topic获取topic的路由信息。另外client有个定时任务，每隔一段时间会从nameserver获取所有该client相关的topic的路由信息，如果新的broker加入，client会有延迟，这个延迟的时间取决于client定时任务拉取的时间。


#### 应对nameserver宕机
client和nameserver的交互只有获取topic的路由信息，一般nameserver有多台提供服务，获取这些信息是从中选择其中一台。broker和nameserver的交互是同步topic的信息（心跳），如果所有的nameserver都当机了才会对系统产生影响。如果所有的nameserver都当机了，新的broker无法加入，因为新的broker加入集群的通知机制是client从nameserver定时拉取。老的client还是可以发送消息到broker。

