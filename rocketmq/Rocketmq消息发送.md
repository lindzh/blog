---
title: RocketMQ消息发送解析
date: 2017-05-08 16:20:03
categories:
- rocketmq
tags:
- rocketmq
- 分布式
- 消息中间件
---

![](http://lindzh.oss-cn-hangzhou.aliyuncs.com/blog/msg_send.png)

#### RocketMQ消息发送涉及到的角色与过程

RocketMQ的消息发送涉及三个角色，NameServer，Broker和Producer：
1. NameServer：保存topic queue路由信息，并提供topic 路由信息api供Client获取。
2. broker：提供消息发送api，并将接收到的消息存储到该topic相应的Queue上。
3. producer：消息发送client。

发送消息的过程如下：

1. client从nameserver获取topic queue路由信息。
2. client根据路由信息生成该topic所有queue列表。
3. client从该topic的queue列表中随机选择一个queue，并将消息发送到Queue上，Queue中有具体Broker的信息和queueid，发送时将消息发送到该Broker并带上queueid即可。



#### client获取topic路由信息
client发送消息的时候会从本地的cache中拿topic的路由信息，如果路由信息没在本地，client会从NameServer获取路由信息，代码如下：

```java
TopicPublishInfo topicPublishInfo = this.topicPublishInfoTable.get(topic);
if (null == topicPublishInfo || !topicPublishInfo.ok()) {
    this.topicPublishInfoTable.putIfAbsent(topic, new TopicPublishInfo());
    this.mQClientFactory.updateTopicRouteInfoFromNameServer(topic);
    topicPublishInfo = this.topicPublishInfoTable.get(topic);
}
```
获取完毕后，该topic的路由信息会保存在本地。

#### 从nameserver获取路由信息

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
##### 路由信息格式

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
一般Topic为无序Topic，无序Topic是没有order conf的，拿到的理由信息是有该topic且角色为Master的Broker列表，这里的order conf为局部有序Topic才会有，局部有序Topic的应用场景如同一个订单的消息需要发送到同一个queue中来保证消息先发先消费的顺序性，如果后发送的消息先到了会给业务造成异常（如订单付款和订单取消）。关于创建顺序topic 请参考mqadmin 创建topic的帮助。

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
对于顺序Topic的场景下，需要用户自己选择想要发送的queue来保证发送的顺序性，Producer也提供了api，如将queue传入send方法，选择send包含MessageQueueSelector的api来选择需要发送的queue。顺序Topic的需求场景：局部要求顺序，如订单支付，订单发货等。

```java
public interface MessageQueueSelector {
    MessageQueue select(final List<MessageQueue> mqs, final Message msg, final Object arg);
}
```


#### Producer提供的发送api

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
如果broker宕机，client能较快感知到broker宕机的是发送到该broker的消息会失败。如果没有消息发送，则需要通过nameserver的自动清理broker的心跳更新定时任务更新topic路由信息，client定时任务拉取到才能感知到。时间预测：nameserver清理broker时间2分钟执行一次，client定时拉取时间默认30s，可配置，所以最多需要2分30秒client才能感知到。Broker 99.5%的可用性，很少会发生宕机，如果发送宕机，磁盘没有损坏的情况下不会有消息丢失。

##### 处理消息发送失败
对于Broker宕机，发送到该Broker的消息会发送失败，默认情况下，producer的client在发送消息的时候发现Broker异常了，会将该Broker标记为异常一段时间（即随机退避），并尝试重新发送消息（重新选择Broker，选择queue发送）。对于无序的Topic，client的重试次数是3次，一般Broker宕机对消息发送无影响，发送到该Broker失败了会发送到别的Broker。对于有序Topic，指定了queue，相当于指定了Broker，此时消息发送会失败，需要业务处理（处理方式如存DB，存本地文件系统等）。

无序消息应对宕机:如果设置了重试次数，消息会重新发送到其他broker。默认重试次数：3

#### 新的broker加入
broker加入到集群或者在broker上面创建了topic，broker会同步该broker上所有的topic信息到nameserver，这也是broker和nameserver的心跳。client发送消息首先会从本地缓存中拿到topic的路由信息，如果本地没有，会从nameserver通过传入topic获取topic的路由信息。另外client有个定时任务，每隔一段时间会从nameserver获取所有该client相关的topic的路由信息，如果新的broker加入，client会有延迟，这个延迟的时间取决于client定时任务拉取的时间。


#### 应对nameserver宕机
client和nameserver的交互只有获取topic的路由信息，一般nameserver有多台提供服务，获取这些信息是从中选择其中一台。broker和nameserver的交互是同步topic的信息（心跳），如果所有的nameserver都宕机了才会对系统产生影响，此时的影响是系统无法扩容，不能创建Topic，不能进行系统运维，已经启动的消息发送与消费无影响，不能启动新的。

