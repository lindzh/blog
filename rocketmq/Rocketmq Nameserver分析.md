---
title: RocketMQ之NameServer
date: 2017-06-08 16:20:03
categories:
- rocketmq
tags:
- rocketmq
- 分布式
- nameserver
---

#### Nameserver在RocketMQ中的作用
NameServer作为RocketMQ的注册中心，承担着Topic路由，Broker发现，其他配置信息保存等核心作用，RocketMQ 2.0版本使用了Zookeeper作为其注册中心，后来更换为自己写的TCP注册中心来解决在上千个Broker实例下Zookeeper通知和性能问题 。

##### 一、KV存储
RocketMQ的KV存储本质上是一个HashMap，存储数据结构如下：
```java
private final HashMap<String/* Namespace */, HashMap<String/* Key */, String/* Value */>> configTable
     = new HashMap<String, HashMap<String, String>>();
```
namespace下存储key，value,用不同的namespace表示不同的业务类型。namespce如`NAMESPACE_ORDER_TOPIC_CONFIG`用来存储顺序Topic的Broker排序。另外KV存储也提供了操作KV的api如增加，删除，修改等操作。

##### 二、Broker注册
Broker启动后会通过定时任务注册到NameServer，这个注册有两个作用：

1. 通过mqadmin命令行工具创建topic到指定的Broker后，这个Broker会在注册到注册中心的时候将Topic注册到NameServer中，这样Producer和Consumer就可以发现Topic的路由信息了。
2. Broker可能会宕机，注册的作用就是告诉NameServer Broker还活着。如果Broker宕机了，Producer和Consumer会把该Broker标记为NotActive，并使用定时任务从NameServer获取Topic的路由信息，Broker宕机会发生路由信息的改变。

![](http://lindzh.oss-cn-hangzhou.aliyuncs.com/blog/nameserver.png)

```java
case RequestCode.REGISTER_BROKER:
    Version brokerVersion = MQVersion.value2Version(request.getVersion());
    if (brokerVersion.ordinal() >= MQVersion.Version.V3_0_11.ordinal()) {
        return this.registerBrokerWithFilterServer(ctx, request);
    } else {
        return this.registerBroker(ctx, request);
    }
```

`Nameserver` 收到`Broker`的注册信息后会保存`broker`所属集群，同时Broker会带上topic相关信息，如果这个Broker之前没有注册过，是新的broker，则会更新这个broker上所带的topic信息到nameserver中。另外，如果broker断线，他的心跳信息会被定时任务清除，会更新这个broker的心跳信息。代码如下：

```java
public RegisterBrokerResult registerBroker(
    final String clusterName,
    final String brokerAddr,
    final String brokerName,
    final long brokerId,
    final String haServerAddr,
    final TopicConfigSerializeWrapper topicConfigWrapper,//topic信息列表 map
    final List<String> filterServerList,
    final Channel channel) {
    RegisterBrokerResult result = new RegisterBrokerResult();
    try {
        try {
            this.lock.writeLock().lockInterruptibly();
			//加入集群
            Set<String> brokerNames = this.clusterAddrTable.get(clusterName);
            if (null == brokerNames) {
                brokerNames = new HashSet<String>();
                this.clusterAddrTable.put(clusterName, brokerNames);
            }
            brokerNames.add(brokerName);

            boolean registerFirst = false;
			//broker地址以及编号信息注册，为0表示master
            BrokerData brokerData = this.brokerAddrTable.get(brokerName);
            if (null == brokerData) {
                registerFirst = true;
                brokerData = new BrokerData();
                brokerData.setBrokerName(brokerName);
                HashMap<Long, String> brokerAddrs = new HashMap<Long, String>();
                brokerData.setBrokerAddrs(brokerAddrs);

                this.brokerAddrTable.put(brokerName, brokerData);
            }
            String oldAddr = brokerData.getBrokerAddrs().put(brokerId, brokerAddr);
            registerFirst = registerFirst || (null == oldAddr);
				//master 第一次注册 更新 queuedata
            if (null != topicConfigWrapper //
                && MixAll.MASTER_ID == brokerId) {
                if (this.isBrokerTopicConfigChanged(brokerAddr, topicConfigWrapper.getDataVersion())//
                    || registerFirst) {
                    ConcurrentHashMap<String, TopicConfig> tcTable =
                        topicConfigWrapper.getTopicConfigTable();
                    if (tcTable != null) {
                        for (Map.Entry<String, TopicConfig> entry : tcTable.entrySet()) {
                            this.createAndUpdateQueueData(brokerName, entry.getValue());
                        }
                    }
                }
            }
				//心跳信息
            BrokerLiveInfo prevBrokerLiveInfo = this.brokerLiveTable.put(brokerAddr,
                new BrokerLiveInfo(
                    System.currentTimeMillis(),
                    topicConfigWrapper.getDataVersion(),
                    channel,
                    haServerAddr));
            if (null == prevBrokerLiveInfo) {
                log.info("new broker registerd, {} HAServer: {}", brokerAddr, haServerAddr);
            }

            if (filterServerList != null) {
                if (filterServerList.isEmpty()) {
                    this.filterServerTable.remove(brokerAddr);
                } else {
                    this.filterServerTable.put(brokerAddr, filterServerList);
                }
            }

            if (MixAll.MASTER_ID != brokerId) {
                String masterAddr = brokerData.getBrokerAddrs().get(MixAll.MASTER_ID);
                if (masterAddr != null) {
                    BrokerLiveInfo brokerLiveInfo = this.brokerLiveTable.get(masterAddr);
                    if (brokerLiveInfo != null) {
                        result.setHaServerAddr(brokerLiveInfo.getHaServerAddr());
                        result.setMasterAddr(masterAddr);
                    }
                }
            }
        } finally {
            this.lock.writeLock().unlock();
        }
    } catch (Exception e) {
        log.error("registerBroker Exception", e);
    }

    return result;
}

```

##### 三、Broker心跳
Broker会定时上传心跳信息，同时Nameserver会启动定时任务10秒钟执行一次来清理没有上传心跳超过一定时间的Broker，被清理的Broker会下线，同时与Nameserver的连接会被关闭。

+ NameServer的清理，清理时间间隔：10s

```java
this.scheduledExecutorService.scheduleAtFixedRate(new Runnable() {

    @Override
    public void run() {
        NamesrvController.this.routeInfoManager.scanNotActiveBroker();
    }
}, 5, 10, TimeUnit.SECONDS);

```
清理代码如下：

```java
public void scanNotActiveBroker() {
    Iterator<Entry<String, BrokerLiveInfo>> it = this.brokerLiveTable.entrySet().iterator();
    while (it.hasNext()) {
        Entry<String, BrokerLiveInfo> next = it.next();
        long last = next.getValue().getLastUpdateTimestamp();
        if ((last + BROKER_CHANNEL_EXPIRED_TIME) < System.currentTimeMillis()) {
            RemotingUtil.closeChannel(next.getValue().getChannel());
            it.remove();
            log.warn("The broker channel expired, {} {}ms", next.getKey(), BROKER_CHANNEL_EXPIRED_TIME);
            this.onChannelDestroy(next.getKey(), next.getValue().getChannel());
        }
    }
}
```
其中destroy方法做的工作如下：

1. 清理broker在集群中的信息
2. 清理该broker上的filter信息
3. 由于RockMQ的Broker由Master和Slave组成，BrokerName相同的为同一个Group，BrokerId为O的为MASTER，Broker Id不为0的为SLAVE，清理该broker时检查与该Broker相同的Group下还有没有别的Broker，如果没有其他Broker了，那说明该Group已经不可用，这时会清理掉这个Group下的topic路由信息，因为该topic在这个Group下没有副本可以发送或者消费消息。

说明：RocketMQ本身没有Group的概念，这里借用相同的BrokerName Master，Slave关系来解释和理清他们之间的关系。

Broker过期心跳过期时间：
BROKER_CHANNEL_EXPIRED_TIME = 1000 * 60 * 2 2分钟

分析：如果一个Broker阻塞或者宕机，没有发送心跳到nameserver，那么最多需要2分钟+client发现时间才能删除。

+ Broker心跳更新

Broker会定时注册到NameServer来更新心跳信息，本质上的心跳包包含了该Broker上所有的Topic列表。

##### 四 NameServer之开放api

```java
public RemotingCommand processRequest(ChannelHandlerContext ctx, RemotingCommand request) throws RemotingCommandException {
        switch (request.getCode()) {
            case RequestCode.PUT_KV_CONFIG:
                return this.putKVConfig(ctx, request);
            case RequestCode.GET_KV_CONFIG:
                return this.getKVConfig(ctx, request);
            case RequestCode.DELETE_KV_CONFIG:
                return this.deleteKVConfig(ctx, request);
            case RequestCode.REGISTER_BROKER:
                Version brokerVersion = MQVersion.value2Version(request.getVersion());
                if (brokerVersion.ordinal() >= MQVersion.Version.V3_0_11.ordinal()) {
                    return this.registerBrokerWithFilterServer(ctx, request);
                } else {
                    return this.registerBroker(ctx, request);
                }
            case RequestCode.UNREGISTER_BROKER:
                return this.unregisterBroker(ctx, request);
            case RequestCode.GET_ROUTEINTO_BY_TOPIC:
                return this.getRouteInfoByTopic(ctx, request);
            case RequestCode.GET_BROKER_CLUSTER_INFO:
                return this.getBrokerClusterInfo(ctx, request);
            case RequestCode.WIPE_WRITE_PERM_OF_BROKER:
                return this.wipeWritePermOfBroker(ctx, request);
            case RequestCode.GET_ALL_TOPIC_LIST_FROM_NAMESERVER:
                return getAllTopicListFromNameserver(ctx, request);
            case RequestCode.DELETE_TOPIC_IN_NAMESRV:
                return deleteTopicInNamesrv(ctx, request);
            case RequestCode.GET_KVLIST_BY_NAMESPACE:
                return this.getKVListByNamespace(ctx, request);
            case RequestCode.GET_TOPICS_BY_CLUSTER:
                return this.getTopicsByCluster(ctx, request);
            case RequestCode.GET_SYSTEM_TOPIC_LIST_FROM_NS:
                return this.getSystemTopicListFromNs(ctx, request);
            case RequestCode.GET_UNIT_TOPIC_LIST:
                return this.getUnitTopicList(ctx, request);
            case RequestCode.GET_HAS_UNIT_SUB_TOPIC_LIST:
                return this.getHasUnitSubTopicList(ctx, request);
            case RequestCode.GET_HAS_UNIT_SUB_UNUNIT_TOPIC_LIST:
                return this.getHasUnitSubUnUnitTopicList(ctx, request);
            case RequestCode.UPDATE_NAMESRV_CONFIG:
                return this.updateConfig(ctx, request);
            case RequestCode.GET_NAMESRV_CONFIG:
                return this.getConfig(ctx, request);
            default:
                break;
        }
        return null;
    }
```
通过阅读源码得知：NameServer提供了操作KV的api，Broker注册（Broker注册其实是心跳上传），Broker取消注册，获取集群Broker列表，获取Topic路由信息等。

##### 五、其他
NameServer配置文件的更新提供了api，可以通过api更新配置，同时配置文件更新了会持久化到文件系统中，初次启动时会使用该配置文件。
