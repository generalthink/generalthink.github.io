---
title: Kafka Consumer源码之Offset以及Fetcher分析
date: 2019-05-31 10:19:44
tags: Kafka
---



上一篇讲了consumer如何加入consumer group的,现在加入组成功之后,就要准备开始消费,但是我们需要知道consumer从offset为多少的位置开始消费。

consumer中关于如何消费有2种策略：

**1. 手动指定**
调用consumer.seek(TopicPartition, offset),然后开始poll

**2. 自动指定**
poll之前给集群发送请求，让集群告知客户端，当前该TopicPartition的offset是多少,这也是我们此次分析的重点.

<!--more-->

在讲如何拉取offset之前,先认识下下面这个类
```java
private static class TopicPartitionState {
  private Long position; // last consumed position
  private Long highWatermark; // the high watermark from last fetch
  private Long logStartOffset; // the log start offset
  private Long lastStableOffset;
  private boolean paused;  // whether this partition has been paused by the user
  private OffsetResetStrategy resetStrategy;  // the strategy to use if the offset needs resetting
  private Long nextAllowedRetryTimeMs;
}
```
consumer实例订阅的每个 topic-partition 都会有一个对应的 TopicPartitionState 对象，在这个对象中会记录上面内容,最需要关注的就是position这个属性,它表示上一次消费的位置。其中通过consumer.seek方式指定消费offset的时候,其实设置的就是这个position值。


加入group之后，就得去获取offset了,下面的方式就是开始更新position(offset)

```java
private boolean updateFetchPositions(final long timeoutMs) {
  
  // 1. 查看TopicPartitionState的position是否为空,第一次消费肯定为空
  cachedSubscriptionHashAllFetchPositions = subscriptions.hasAllFetchPositions();
  if (cachedSubscriptionHashAllFetchPositions) return true;

  // 2. 如果没有有效的offset,那么需要从group coordinator中获取
  if (!coordinator.refreshCommittedOffsetsIfNeeded(timeoutMs)) return false;

  // 3. 如果还存在partition不知道position,并且设置了offsetreset策略,那么就等待重置，不然就抛出异常
  subscriptions.resetMissingPositions();

  // 4. 向PartitionLeader(GroupCoordinator所在机器)发送ListOffsetRequest重置position
  fetcher.resetOffsetsIfNeeded();

  return true;
}
```

上面的代码主要分为4个步骤,具体如下:
1. 首先查看当前TopicPartition的position是否为空，如果不为空，表示知道下次fetch position(拉取数据从哪个位置开始拉取),
但是第一次消费这个 TopicPartitionState.position肯定为空。

2. 通过coordinator为缺少fetch position的partition拉取position(last committed offset)

3. 仍不知道partition的position(_consumer_offsets中未保存位移信息),且设置了offsetreset策略,那么就等待重置，
如果没有设置重置策略，然就抛出NoOffsetForPartitionException异常。

4. 为那些需要重置fetch position的partition发送ListOffsetRequest重置position(consumer.beginningOffsets(),consumer.endOffsets(),consumer.offsetsForTimes(),consumer.seek()
都会发送ListOffRequest请求)
>上面说的几个方法相当于都是用户自己自定义消费的offset,所以可能出现越界(消费位置无法在实际分区中查到)的情况，所以也是也是会发送ListOffsetRequest请求的，即触发auto.offset.reset参数的执行
>比如现在某个partitiion的可拉取offset最大值为100,如果你指定消费offset=200的位置,那肯定拉取不到，此时就会根据auto.offset.reset策略将拉取位置重置为100(默认为latest)


我们先看下第二个步骤是如何fetch position的

```java
 public Map<TopicPartition, OffsetAndMetadata> fetchCommittedOffsets(final Set<TopicPartition> partitions,
                                                                        final long timeoutMs) {
  while (true) { 
    // 1. 封装FetchRequest请求
    future = sendOffsetFetchRequest(partitions);

    // 2. 通过KafkaClient发送
    client.poll(future, remainingTimeAtLeastZero(timeoutMs, elapsedTime));
    final RequestFuture<Map<TopicPartition, OffsetAndMetadata>> future;

    // 3. 获取请求的响应数据
    if (future.succeeded()) {
      return future.value();
    } 
  }
}

```
上面的步骤和我们之前提到的发送其他请求毫无区别,基本就是这三个套路，在获取到响应之后，会设置TopicPartition的position值

```java
public boolean refreshCommittedOffsetsIfNeeded(final long timeoutMs) {
  final Set<TopicPartition> missingFetchPositions = subscriptions.missingFetchPositions();

  final Map<TopicPartition, OffsetAndMetadata> offsets = fetchCommittedOffsets(missingFetchPositions, timeoutMs);
  if (offsets == null) return false;

  for (final Map.Entry<TopicPartition, OffsetAndMetadata> entry : offsets.entrySet()) {
      final TopicPartition tp = entry.getKey();

      // 获取response中的offset
      final long offset = entry.getValue().offset();
      log.debug("Setting offset for partition {} to the committed offset {}", tp, offset);

      // 实际就是设置SubscriptionState的position值
      this.subscriptions.seek(tp, offset);
  }
  return true;
}
```
通过subscriptions.seek()方法为每个TopicPartition设置position值后,到了这里就知道从哪里开发消费订阅topic下的partition了。


第三个步骤什么时候发起FetchRequest拿不到position呢?

我们知道消费位移(consume offset)是保存在_consumer_offsets这个topic里面的,当我们进行消费的时候需要知道上次消费到了什么位置。
那么就会发起请求去看上次消费到了topic的partition的哪个位置，但是这个消费位移是有保存时长的,默认为7天(broker端通过
offsets.retention.minutes)。

之后隔了一段时间进行消费，这个这个间隔时间超过参数的配置值，那么原先的位移信息就会丢失，最后只能通过客户端参数auto.offset.reset来确定开始消费的位置。

如果我们第一次消费topic,那么在_consumer_offsets中也是找不到消费位移的，所以就会执行第四个步骤，发起ListOffsetRequest请求根据配置的reset策略(auto.offset.reset)来决定开始消费的位置。

发起这个请求和之前的请求没有什么特殊之处,都是组装请求体，然后调用poll发起请求，最后处理response,处理response的核心代码如下:

```java
future.addListener(new RequestFutureListener<ListOffsetResult>() {
 @Override
 public void onSuccess(ListOffsetResult result) {
    
     for (Map.Entry<TopicPartition, OffsetData> fetchedOffset : result.fetchedOffsets.entrySet()) {
         TopicPartition partition = fetchedOffset.getKey();
         OffsetData offsetData = fetchedOffset.getValue();
         Long requestedResetTimestamp = resetTimestamps.get(partition);
         resetOffsetIfNeeded(partition, requestedResetTimestamp, offsetData);
     }
 }
}

private void resetOffsetIfNeeded(TopicPartition partition, Long requestedResetTimestamp, OffsetData offsetData) {
  if (!subscriptions.isAssigned(partition)) {
      log.debug("Skipping reset of partition {} since it is no longer assigned", partition);
  } else if (!subscriptions.isOffsetResetNeeded(partition)) {
      log.debug("Skipping reset of partition {} since reset is no longer needed", partition);
  } else if (!requestedResetTimestamp.equals(offsetResetStrategyTimestamp(partition))) {

      // 如果reset策略设置的是latest，那么requestedResetTimestamp = -1，如果是earliest,requestedResetTimestamp = -2
      log.debug("Skipping reset of partition {} since an alternative reset has been requested", partition);
  } else {
      log.info("Resetting offset for partition {} to offset {}.", partition, offsetData.offset);

      // 设置对应的TopicPartition fetch的position
      subscriptions.seek(partition, offsetData.offset);
  }
}
```

这里解释下auto.offset.reset的两个值(latest,earliest)的区别,假设我们现在要消费MyConsumerTopic的数据,它有3个分区,生产者往这个topic发送了10条数据,然后分区数据按照MyConsumerTopic-0(3条数据),MyConsumerTopic-1(3条数据),MyConsumerTopic-2(4条数据)这样分配。

当设置为latest的时候,返回的offset具体到每个partition就是HW值(partition0就是3,partition1也是3,partition2是4)

当设置为earliest的时候,就会从起始处(LogStartOffset,不是LSO)开始消费，这里就是从0开始

![名词解析](/images/introduction-kafka/kafka-partition-analysis.png)

1. **LogStartOffset:**表示partition的起始位置,初始值为0,由于消息的增加以及日志清除策略影响，这个值会阶段性增大。尤其注意这个不能缩写未LSO,LSO代表的是LastStableOffset,和事务有关。
2. **ConsumerOffset:**消费位移，表示partition的某个消费者消费到的位移位置。
3. **HighWatermark**: 简称HW,代表消费端能看到的partition的最高日志位移,HW大于等于ConsumerOffset的值。
4. **LogEndOffset**: 简称LEO,代表partition的最高日志位移，第消费者不可见,HW到LEO这之间的数据未被follwer完全同步。


现在万事俱备只欠东风了,加入组，也确定了拉取的offset,那么现在就应该去拉取数据了,其核心源码如下:


```java
private Map<TopicPartition, List<ConsumerRecord<K, V>>> pollForFetches(final long timeoutMs) {
   ...

  // 获取fetcher已经拉取到的数据
  final Map<TopicPartition, List<ConsumerRecord<K, V>>> records = fetcher.fetchedRecords();
  if (!records.isEmpty()) {
      return records;
  }

  // 上次fetch到的数据已经全部拉取了,需要再次发送fetch请求,从broker拉取数据

  // 组装请求,将发往同一个节点的请求放到一起
  fetcher.sendFetches();

  // 真正开始发送,底层同样使用NIO
  client.poll(pollTimeout, startMs, () -> {
     return !fetcher.hasCompletedFetches();
  });

  // 如果 group 需要 rebalance,直接返回空数据,这样更快地让 group 进行稳定状态
  if (coordinator.rejoinNeededOrPending()) {
     return Collections.emptyMap();
  }

  // 获取拉取到的数据
  return fetcher.fetchedRecords();
 }

```

这里需要注意的是fetcher.sendFetches(),在发送请求的同时会注册回调函数，当有response的时候，会解析response，讲返回的数据放到Fetcher的成员变量中

```java
client.send(fetchTarget, request)
  .addListener(new RequestFutureListener<ClientResponse>() {
    @Override
    public void onSuccess(ClientResponse resp) {
        FetchResponse<Records> response = (FetchResponse<Records>) resp.responseBody();
        
        Set<TopicPartition> partitions = new HashSet<>(response.responseData().keySet());
        FetchResponseMetricAggregator metricAggregator = new FetchResponseMetricAggregator(sensors, partitions);

        for (Map.Entry<TopicPartition, FetchResponse.PartitionData<Records>> entry : response.responseData().entrySet()) {
            TopicPartition partition = entry.getKey();
            long fetchOffset = data.sessionPartitions().get(partition).fetchOffset;
            FetchResponse.PartitionData fetchData = entry.getValue();

            // 将返回的数据放到ConcurrentLinkedQueue<CompletedFetch>
            completedFetches.add(new CompletedFetch(partition, fetchOffset, fetchData, metricAggregator,
                    resp.requestHeader().apiVersion()));
        }
    }
  });
}
```

剩下的获取拉取到的数据我就不做过多分析了,以免有凑字数嫌疑,至此Kafka Consumer中如何拉取消息的整体流程分析完毕,接下来的文章会分析partition的分配以及offset的提交，敬请期待。