##Topic创建解析
topic的创建有两种方式，一种方式是开启client创建模式，另一种是使用mqadmin命令行创建topic，这两种方式中无论是哪种方式，都需要指定broker或者cluster创建，如果是指定broker创建，则该topic只在指定的broker列表上存在，如果指定cluster则在该cluster下的多个broker上都存在。

##### 参数说明
```java
public class TopicConfig {
    private static final String SEPARATOR = " ";
    public static int defaultReadQueueNums = 16;
    public static int defaultWriteQueueNums = 16;
    private String topicName;
    private int readQueueNums = defaultReadQueueNums;
    private int writeQueueNums = defaultWriteQueueNums;
    private int perm = PermName.PERM_READ | PermName.PERM_WRITE;
    private TopicFilterType topicFilterType = TopicFilterType.SINGLE_TAG;
    private int topicSysFlag = 0;
    private boolean order = false;
}
```
读写queue的数量默认是8，是通过mqadmin代码指定的，如下所示：

```java
TopicConfig topicConfig = new TopicConfig();
topicConfig.setReadQueueNums(8);
topicConfig.setWriteQueueNums(8);
```
集群模式中每个broker中的readqueue和writequeue数量都一样，单个broker指定只有该broker中有这个topic。

perm参数：

order参数：表示顺序，如果有顺序，例如局部顺序，全局有序，则需要指定order为true，默认无序。order有序会在nameserver的KV中保存topic的queue在broker列表的顺序

topicSysFlag参数：表示是否是unit或者centersync，如下：

```java
int topicCenterSync = TopicSysFlag.buildSysFlag(isUnit, isCenterSync);
topicConfig.setTopicSysFlag(topicCenterSync);
```

###创建过程
根据创建指定broker和不指定broker指定cluster分为两种方式创建，指定Topic，指定集群

#####通过指定broker创建
通过broker创建的方式，可以指定readQueueNumbers和writeQueueNumbers，如果没有指定这两个参数，则默认为8。可以分别创建topic在不同的broker上，client收到topic的路由信息后会发到多个broker。如果是有序topic，则会把broker的顺序发送到nameserver上，变为多个broker有顺序。

##### 通过指定集群的方式创建
通过集群创建topic，先通过集群名称从nameserver拿到broker的列表，然后再在各个broker上创建topic，如果是有序topic，则会更新broker的顺序和队列数量到nameserver中，先后顺序代表了消息发送的顺序。

##### Nameserver中topic broker的顺序
brokerName:writeQueueNumbers;brokerName:writeQueueNumbers