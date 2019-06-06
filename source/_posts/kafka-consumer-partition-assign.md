---
title: Kafka Consumer源码分析之分区方案
date: 2019-06-06 14:39:56
tags: Kafka
---
consumer提供三种不同的分区策略，可以通过`partition.assignment.strategy`参数进行配置,默认使用的策略是`org.apache.kafka.clients.consumer.RangeAssignor`,还存在`org.apache.kafka.clients.consumer.RoundRobinAssignor`和`org.apache.kafka.clients.consumer.StickyAssignor`这两种，它们的关系图如下所示。

![partition分配机制](/images/introduction-kafka/kafka-partition-assignor.png)

<!--more-->

当我们想要自定义partition分配策略的时候只需要继承`AbstractPartitionAssignor`这个类就行了。


### AbstractPartitionAssignor

这个抽象类有一个抽象方法，其他子类都是通过复写它的抽象方法来实现分区分配的
```java
public abstract Map<String, List<TopicPartition>> assign(Map<String, Integer> partitionsPerTopic,
                                                             Map<String, Subscription> subscriptions);

```
assign() 这个方法，有两个参数：

1. partitionsPerTopic：所订阅的每个 topic 与其 partition 数的对应关系，metadata 没有的 topic 将会被移除
2. subscriptions：每个 consumerId 与其所订阅的 topic 列表的关系。

### RangeAssignor 分区分配

先看下这个策略的分区代码实现：
```java
@Override
public Map<String, List<TopicPartition>> assign(Map<String, Integer> partitionsPerTopic,
                                             Map<String, Subscription> subscriptions) {
  // 1. 获取每个topic被多少个consumer订阅了
 Map<String, List<String>> consumersPerTopic = consumersPerTopic(subscriptions);
  // 2. 存储最终的分配方案
 Map<String, List<TopicPartition>> assignment = new HashMap<>();
 for (String memberId : subscriptions.keySet())
     assignment.put(memberId, new ArrayList<TopicPartition>());

 for (Map.Entry<String, List<String>> topicEntry : consumersPerTopic.entrySet()) {
     String topic = topicEntry.getKey();
     List<String> consumersForTopic = topicEntry.getValue();

     // 3. 每个topic的partition数量
     Integer numPartitionsForTopic = partitionsPerTopic.get(topic);
     if (numPartitionsForTopic == null)
         continue;

     Collections.sort(consumersForTopic);

     // 4. 表示平均每个consumer会分配到多少个partition
     int numPartitionsPerConsumer = numPartitionsForTopic / consumersForTopic.size();
     // 5. 平均分配后还剩下多少个partition未被分配
     int consumersWithExtraPartition = numPartitionsForTopic % consumersForTopic.size();

     List<TopicPartition> partitions = AbstractPartitionAssignor.partitions(topic, numPartitionsForTopic);

     // 6. 这里是关键点,分配原则是将未能被平均分配的partition分配到前consumersWithExtraPartition个consumer
     for (int i = 0, n = consumersForTopic.size(); i < n; i++) {
         int start = numPartitionsPerConsumer * i + Math.min(i, consumersWithExtraPartition);
         int length = numPartitionsPerConsumer + (i + 1 > consumersWithExtraPartition ? 0 : 1);
         assignment.get(consumersForTopic.get(i)).addAll(partitions.subList(start, start + length));
     }
 }
 return assignment;
}
```

上面最重要的就是第六步，它决定了如何分配的具体方案，其分配规则是:先将分区数平均分配给consumer，对于剩下不能被平均分配的partition，会将其分配到前 consumersWithExtraPartition 个 consumer 上，也就是前 consumersWithExtraPartition 个 consumer 获得 topic-partition 列表会比后面多一个。

举个例子,假设一个topic有5个partiton,然后一个group中有3个consumer都订阅了这个topic,那么range的分配方式如下

+ consumer 0: start-->0, length-->2, topic-partition-->p0,p1
+ consumer 1: start-->2, length-->2, topic-partition-->p2,p3
+ consumer 2: start-->4, length-->1, topic-partition-->p4

如果group中有consumer没有订阅这个topic，那么就不会参与分配，对于多个topic的分配方案,和单个topic的分配是一样的,同样的再举个例子。

现在有2个topic,一个partition有3个,一个partition有5个,group中有3个consumer，但是只有前面2个consumer订阅了第一个topic,而另外一个topic则被所有consumer都订阅了，那么其分配方案取下：

| consumer  | 订阅topic1 | 订阅topci2  |
|---|---|---|
| consumer0  | t1-p0, t1-p1  | t2-p0, t2-p1  |
| consumer1  | t1-p2  | t2-p2, t2-p3  |
| consumer2  |   | t2-p4  |


其实也可以看到，随着分区数的变化,这种方式的分配并不均匀，类似的情况如果扩大，则部分消费者可能会消费很多partition，而部分消费者又会闲置。

### RoundRobinAssignor 分区分配

同样的还是想来看下它的源代码:
```java
public Map<String, List<TopicPartition>> assign(Map<String, Integer> partitionsPerTopic,
                                                    Map<String, Subscription> subscriptions) {
  Map<String, List<TopicPartition>> assignment = new HashMap<>();
  for (String memberId : subscriptions.keySet())
      assignment.put(memberId, new ArrayList<TopicPartition>());
  // 1. 环状链表,存储所有的consumer,一次迭代完之后又会回到原点
  CircularIterator<String> assigner = new CircularIterator<>(Utils.sorted(subscriptions.keySet()));
  // 2. 获取所有订阅的topic的partition总数
  for (TopicPartition partition : allPartitionsSorted(partitionsPerTopic, subscriptions)) {
      final String topic = partition.topic();
      while (!subscriptions.get(assigner.peek()).topics().contains(topic))
          assigner.next();
      assignment.get(assigner.next()).add(partition);
  }
  return assignment;
}
```

其分配规则很简单列出所有的topic-partition以及所有的consumer,然后开始分配,先每个consumer都分配一轮,一轮分配完成之后接着下一轮继续分配，直到分配完为止。
同样的举个例子,假设一个topic有5个partiton,然后一个group中有3个consumer都订阅了这个topic,那么roundrobin的分配方式如下；
+ consuemr 0: topic-partition-->p0,p3
+ consumer 1: topic-partition-->p1,p4
+ consumer 2: topic-partition-->p2

对于多个consumer订阅多个topic的情况，这里也举一个例子说明

现在有3个topic,一个有2个partition,一个有3个partition,另外一个有4个partition,group中有3个consumer,第一个consumer订阅了第一个topic,第二个consumer订阅了前两个topic,第三个consumer订阅了三个topic，那么它们的分配方案如下：

| consumer  | topic1  | topic2  | topic3  |
|---|---|---|---|
| consumer1  | t1-p0  |   |   |
| consumer2  | t1-p1  | t2-p0, t2-p3  |   |
| consumer3  |  | t2-p1  | t3-p0, t3-p1, t3-p2, t3-p3  |

很明显consumer3要是把t2-p1也分配给consumer2就会显得更加均匀一些了。


### StickyAssignor 分区分配

sticky这样的分区策略是从0.11版本才开始引入的，它主要有两个目的
1. 分区的分配要尽可能均匀
2. 分区的分配要尽可能与上次分配的保持相同

当两者冲突的时候，第一个目标优先于第二个目标。

源码我就不贴出来了，因为这个类的源码是上面两个类的源码的10倍。

sticky这样的分区方式作用发生分区重分配的时候，尽可能地让前后两次分配相同，进而减少系统资源的损耗及其他异常情况的发生。以上面那个例子举例，如果采用这种分配方式那么其分配结果如下：

| consumer  | topic1  | topic2  | topic3  |
|---|---|---|---|
| consumer1  | t1-p0  |   |   |
| consumer2  | t1-p1, t2-p1  | t2-p0, t2-p3  |   |
| consumer3  |  |   | t3-p0, t3-p1, t3-p2, t3-p3  |


上面三个分区策略有着不同的分配方式，在实际使用过程中，需要根据自己的需求选择合适的策略，但是如果你只有一个consumer,那么选择哪个方式都是一样的，但是如果是多个consumer不在同一台设备上进行消费，那么sticky方式应该更加合适。

#### 自定义分区策略

如之前所说，只需要继承AbstractPartitionAssignor并复写其中方法即可(当然也可以直接实现PartitionAssignor接口),其中有两个方法需要复写

```java
public String name()

public Map<String, List<TopicPartition>> assign(Map<String, Integer> partitionsPerTopic,Map<String, Subscription> subscriptions)

```

其中assign()方法表示的是分区分配方案的实现，而name()方法则表示了这个分配策略的唯一名称，比如之前提到的range,roundrobin和sticky,这个名字会在和GroupCoordinator的通信中返回，通过它consumer leader来确定整个group的分区方案(分区策略是由group中的consumer共同投票决定的，谁使用的多,就是用哪个策略),对于加入group的分析可以参考之前的[文章](https://generalthink.github.io/2019/05/15/how-to-join-kafka-consumer-group)