---
title: Kafka Consumer消费数据你要考虑哪些问题?
date: 2019-05-06 15:59:36
tags: Kafka
---


### 如何消费数据

我们已经知道了如何发送数据到Kafka,既然有数据发送,那么肯定就有数据消费,消费者也是Kafka整个体系中不可缺少的一环

```java
public class KafkaConsumerDemo {
public static void main(String[] args) throws InterruptedException {
    Properties props = new Properties();

    // 必须设置的属性
    props.put("bootstrap.servers", "192.168.239.131:9092");
    props.put("key.deserializer", "org.apache.kafka.common.serialization.StringDeserializer");
    props.put("value.deserializer", "org.apache.kafka.common.serialization.StringDeserializer");
    props.put("group.id", "group1");
    
    // 可选设置属性
    props.put("enable.auto.commit", "true");
    // 自动提交offset,每1s提交一次
    props.put("auto.commit.interval.ms", "1000");
    props.put("auto.offset.reset","earliest ");
    props.put("client.id", "zy_client_id");
    KafkaConsumer<String, String> consumer = new KafkaConsumer<>(props);
    // 订阅test1 topic
    consumer.subscribe(Collections.singletonList("test1"));

    while(true) {
        ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(100));
        records.forEach(record -> {
            System.out.printf("topic = %s ,partition = %d,offset = %d, key = %s, value = %s%n", record.topic(), record.partition(),
                    record.offset(), record.key(), record.value());
        });
    }
  }
}
```
<!--more-->
#### push 还是 pull
Kafka Consumer采用的是主动拉取broker数据进行消费的。一般消息中间件存在推送(server推送数据给consumer)和拉取(consumer主动取服务器取数据)两种方式，这两种方式各有优劣。

如果是选择推送的方式最大的阻碍就是服务器不清楚consumer的消费速度，如果consumer中执行的操作又是比较耗时的，那么consumer可能会不堪重负,甚至会导致系统挂掉。

而采用拉取的方式则可以解决这种情况，consumer根据自己的状态来拉取数据,可以对服务器的数据进行延迟处理。但是这种方式也有一个劣势就是服务器没有数据的时候可能会一直轮询，不过还好Kafka在poll()有参数允许消费者请求在“长轮询”中阻塞，等待数据到达(并且可选地等待直到给定数量的字节可用以确保传输大小)。

#### 必须属性
上面代码中消费者必须的属性有4个,这里着重说一下group.id这个属性,kafka Consumer和Producer不一样,Consummer中有一个Consumer group(消费组)，由它来决定同一个Consumer group中的消费者具体拉取哪个partition的数据,所以这里必须指定group.id属性。

1. bootstrap.servers
  连接Kafka集群的地址，多个地址以逗号分隔
2. key.deserializer
  消息中key反序列化类,需要和Producer中key序列化类相对应
3. value.deserializer
  消息中value的反序列化类,需要和Producer中Value序列化类相对应
4. group.id
  消费者所属消费组的唯一标识

#### 订阅/取消主题

1. 使用subscribe()方法订阅主题
2. 使用assign()方法订阅确定主题和分区
```java
List<PartitionInfo> partitionInfoList = consumer.partitionsFor("topic1");
if(null != partitionInfoList) {
  for(PartitionInfo partitionInfo : partitionInfoList) {
      consumer.assign(Collections.singletonList(
        new TopicPartition(partitionInfo.topic(), partitionInfo.partition())));
  }
}
```
通过subscribe()方法订阅主题具有消费者自动再均衡(reblance)的功能，存在多个消费者的情况下可以根据分区分配策略来自动分配各个消费者与分区的关系。当组内的消费者增加或者减少时，分区关系会自动调整。实现消费负载均衡以及故障自动转移。使用assign()方法订阅则不具有该功能。


3. 取消主题
```java
consumer.unsubscribe();
consumer.subscribe(new ArrayList<>());
consumer.assign(new ArrayList<TopicPartition>());
```
上面的三行代码作用相同，都是取消订阅，其中unsubscribe()方法即可以取消通过subscribe()方式实现的订阅，还可以取消通过assign()方式实现的订阅。

### 如何更好的消费数据

开头处的代码展示了我们是如何消费数据的,但是代码未免过于简单,我们测试的时候这样写没有问题,但是实际开发过程中我们并不会这样写，我们会选择更加高效的方式,这里提供两种方式供大家参考。

1. 一个Consumer group,多个consumer,数量小于等于partition的数量

![多个consumer](/images/introduction-kafka/kafka_multi_consumer.png)

2. 一个consumer,多线程处理事件

![多事件处理器](/images/introduction-kafka/kafka_multi_event_handler.png)

第一种方式每个consumer都要维护一个独立的TCP连接，如果分区数和创建consumer线程的数量过多，会造成不小系统开销。但是如果处理消息足够快速，消费性能也会提升,如果慢的话就会导致消费性能降低。

第二种方式是采用一个consumer，多个消息处理线程来处理消息，其实在生产中，瓶颈一般是集中在消息处理上的(可能会插入数据到数据库，或者请求第三方API)，所以我们采用多个线程来处理这些消息。

当然可以结合第一二种方式，采用多consumer+多个消息处理线程来消费Kafka中的数据,核心代码如下:

```java
for (int i = 0; i < consumerNum; i++) {

  //根据属性创建Consumer
  final Consumer<String, byte[]> consumer = consumerFactory.getConsumer(getServers(), groupId);
  consumerList.add(consumer);

  //订阅主题
  consumer.subscribe(Arrays.asList(this.getTopic()));

  //consumer.poll()拉取数据
  BufferedConsumerRecords bufferedConsumerRecords = new BufferedConsumerRecords(consumer);

  getExecutor().scheduleWithFixedDelay(() -> {
      long startTime = System.currentTimeMillis();

      //进行消息处理
      consumeEvents(bufferedConsumerRecords);

      long sleepTime = intervalMillis - (System.currentTimeMillis() - startTime);
      if (sleepTime > 0) {
        Thread.sleep(sleepTime);
      }
  }, 0, 1000, TimeUnit.MILLISECONDS);
}

```
不过这种方式不能顺序处理数据，如果你的业务是顺序处理，那么第一种方式可能更适合你。所以实际生产中请根据业务选择最适合自己的方式。


### 消费数据考虑哪些问题?

在Kafka中无论是producer往topic中写数据,还是consumer从topic中读数据,都避免不了和offset打交道,关于offset主要有以下几个概念。

![Kafka Offset](/images/introduction-kafka/kafka-partition-offset.png)

+ Last Committed Offset：consumer group最新一次 commit 的 offset，表示这个 group 已经把 Last Committed Offset 之前的数据都消费成功了。
+ Current Position：consumer group 当前消费数据的 offset，也就是说，Last Committed Offset 到 Current Position 之间的数据已经拉取成功，可能正在处理，但是还未 commit。
+ Log End Offset(LEO)：记录底层日志(log)中的**下一条消息的 offset。**,对producer来说，就是即将插入下一条消息的offset。
+ High Watermark(HW)：已经成功备份到其他 replicas 中的最新一条数据的 offset，也就是说 Log End Offset 与 High Watermark 之间的数据已经写入到该 partition 的 leader 中，但是还未完全备份到其他的 replicas 中，consumer是无法消费这部分消息(未提交消息)。

每个Kafka副本对象都有两个重要的属性：LEO和HW。注意是所有的副本，而不只是leader副本。关于这两者更详细解释，建议参考[这篇文章](https://www.cnblogs.com/huxi2b/p/7453543.html)。

对于消费者而言，我们更多时候关注的是消费完成之后如何和服务器进行消费确认，告诉服务器这部分数据我已经消费过了。

这里就涉及到了2个offset，一个是current position,一个是处理完毕向服务器确认的committed offset。显然,异步模式下committed offset是落后于current position的。如果consumer挂掉了,那么下一次消费数据又只会从committed offset的位置拉取数据，就会导致数据被重复消费。

#### 提交策略如何选择

Kafka提供了3种提交offset的方式

1. 自动提交

```java
// 自动提交,默认true
props.put("enable.auto.commit", "true");
// 设置自动每1s提交一次
props.put("auto.commit.interval.ms", "1000");

```

2. 手动同步提交offset

```java
consumer.commitSync();
```

3. 手动异步提交offset

```java
consumer.commitAsync();

```

上面说了既然异步提交offset可能会重复消费,那么我使用同步提交是否就可以表明这个问题呢？我只能说too young,too sample。

```java
while(true) {
  ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(100));
  records.forEach(record -> {
      insertIntoDB(record);
      consumer.commitSync();
  });
}
```

很明显不行,因为insertIntoDB和commitSync()做不到原子操作,如果insertIntoDB()成功了，但是提交offset的时候consumer挂掉了，然后服务器重启，仍然会导致重复消费问题。


#### 是否需要做到不重复消费？

只要保证处理消息和提交offset得操作是原子操作，就可以做到不重复消费。我们可以自己管理committed offset,而不让kafka来进行管理。

比如如下使用方式:

1. 如果消费的数据刚好需要存储在数据库，那么可以把offset也存在数据库，就可以就可以在一个事物中提交这两个结果，保证原子操作。
2. 借助搜索引擎，把offset和数据一起放到索引里面，比如Elasticsearch

每条记录都有自己的offset,所以如果要管理自己的offset还得要做下面事情
1. 设置enable.auto.commit=false
2. 使用每个ConsumerRecord提供的offset来保存消费的位置。
3. 在重新启动时使用seek(TopicPartition, long)恢复上次消费的位置。

通过上面的方式就可以在消费端实现"Exactly Once"的语义,即保证只消费一次。但是是否真的需要保证不重复消费呢？这个得看具体业务,重复消费数据对整体有什么影响在来决定是否需要做到不重复消费。

#### 再均衡(reblance)怎么办？

再均衡是指分区的所属权从一个消费者转移到另一个消费者的行为，再均衡期间，消费组内的消费组无法读取消息。为了更精确的控制消息的消费，我们可以再订阅主题的时候，通过指定监听器的方式来设定发生再均衡动作前后的一些准备或者收尾的动作。

```java
consumer.subscribe(Collections.singletonList("test3"), new ConsumerRebalanceListener() {
  @Override
  public void onPartitionsAssigned(Collection<TopicPartition> partitions) {
      //再均衡之前和消费者停止读取消息之后被调用
  }

  @Override
  public void onPartitionsRevoked(Collection<TopicPartition> partitions) {
      //重新分配分区之后和消费者开始消费之前被调用
  }
});

```

具体如何做得根据具体的业务逻辑来实现,如果消息比较重要，你可以在再均衡的时候处理offset,如果不够重要，你可以什么都不做。

#### 无法消费的数据怎么办?

可能由于你的业务逻辑有些数据没法消费这个时候怎么办？同样的还是的看你认为这个数据有多重要或者多不重要，如果重要可以记录日志,把它存入文件或者数据库，以便于稍候进行重试或者定向分析。如果不重要就当做什么事情都没有发生好了。

### 实际开发中我的处理方式

我开发的项目中,用到kafka的其中一个地方是消息通知(谁给你发了消息,点赞,评论等),大概的流程就是用户在client端做了某些操作，就会发送数据到kafka,然后把这些数据进行一定的处理之后插入到HBase中。

其中采用了 N consumer thread + N Event Handler的方式来消费数据,并采用自动提交offset。对于无法消费的数据往往只是简单处理下，打印下日志以及消息体(无法消费的情况非常非常少)。

得益于HBase的多version控制,即使是重复消费了数据也无关紧要。这样做没有去避免重复消费的问题主要是基于以下几点考虑
1. 重复消费的概率较低，服务器整体性能稳定
2. 即便是重复消费了数据,入库了HBase,获取数据也是只有一条,不影响结果的正确性
3. 有更高的吞吐量
4. 编程简单，不用单独去处理以及保存offset


### 几个重要的消费者参数

+ fetch.min.bytes

  配置poll()拉取请求过程种能从Kafka拉取的最小数据量，如果可用数据量小于它指定的大小会等到有足够可用数据时才会返回给消费者，其默认值时1B

+ fetch.max.wait.ms

  和fetch.min.bytes有关,用于指定Kafka的等待时间，默认时间500ms。如果fetch.min.bytes设置为1MB,fetch.max.wait.ms设置为100ms,Kafka收到消费者请求后,要么返回1MB数据,要么在100ms后返回所有可用数据,就看哪个提交得到满足。

+ max.poll.records

  用于控制单次调用poll()能返回的最大记录数量，默认为500条数据

+ partition.assignment.stragety

  分区会被分配给群组的消费者,这个参数用于指定分区分配策略。默认是RangeAssignore,可选的还有RoundRobinAssignor。同样它还支持自定义


其他更多参数请参考官方文档。