---
title: Kafka consumer的offset的提交方式
date: 2019-06-04 10:18:47
tags: Kafka
---
Kafka consumer的offset提交机制有以下两种

### 手动提交

#### 同步提交
consumer.commitSync()方式提交

#### 异步提交

consumer.commitAsync(callback)方式提交

### 自动提交
props.put("enable.auto.commit", "true");
props.put("auto.commit.interval.ms", "1000");

通过上面启动自动提交以及设置自动提交间隔时间(默认为5s)

<!--more-->

### 源码分析

#### 同步提交源码分析

同步提交的核心代码在ConsumerCoordinator.commitOffsetSync中,核心代码如下:

```java
public boolean commitOffsetsSync(Map<TopicPartition, OffsetAndMetadata> offsets, long timeoutMs) {

  invokeCompletedOffsetCommitCallbacks();
do {
  
  // 1. 组装OffsetCommitRequest
  RequestFuture<Void> future = sendOffsetCommitRequest(offsets);
  // 2. 发送请求
  client.poll(future, remainingMs);
  // 3. 接受到响应之后通知拦截器
  if (future.succeeded()) {
      if (interceptors != null)
          interceptors.onCommit(offsets);
      return true;
  }
  if (future.failed() && !future.isRetriable())
      throw future.exception();
  // 4. 如果请求未完成则休眠一段时间,默认是300毫秒
  time.sleep(retryBackoffMs);

  now = time.milliseconds();
  remainingMs = timeoutMs - (now - startMs);
} while (remainingMs > 0);


return false;

}

```
通过常见的几个步骤就向GroupCoordinator发送了offset的同步信息,同时也完成了offset的confirm
1. 组装OffsetCommitRequest,准备向GroupCoordinator发送请求
2. 发送请求,底层仍然是NIO
3. future返回后调用拦截器
4. 如果响应还未返回则继续等待一段时间,默认是300ms

#### 异步提交源码分析

异步提交的核心源码在ConsumerCoordinator.commitOffsetsAsync中
```java
public void commitOffsetsAsync(final Map<TopicPartition, OffsetAndMetadata> offsets, final OffsetCommitCallback callback) {
  
  // 1. 这里会处理回调
  invokeCompletedOffsetCommitCallbacks();

  // 2. 封装请求
  RequestFuture<Void> future = sendOffsetCommitRequest(offsets);
  final OffsetCommitCallback cb = callback == null ? defaultOffsetCommitCallback : callback;
  // 3. 添加监听器,如果发送完成则处理回调
  future.addListener(new RequestFutureListener<Void>() {
      @Override
      public void onSuccess(Void value) {
          if (interceptors != null)
              interceptors.onCommit(offsets);

          completedOffsetCommits.add(new OffsetCommitCompletion(cb, offsets, null));
      }

      @Override
      public void onFailure(RuntimeException e) {
          Exception commitException = e;

          if (e instanceof RetriableException)
              commitException = new RetriableCommitFailedException(e);

          completedOffsetCommits.add(new OffsetCommitCompletion(cb, offsets, commitException));
      }
  });

```

看到上面的三个步骤你是否会疑惑,怎么没有poll发送数据呢?其实异步提交offset时发送commitOffsetRequest是在consumer.poll()中完成的,这个时候会把Channel中堆积的数据发送出去。
同时同步发送和异步发送还有一个阻塞的区别，也就是上面同步发送未收到响应的时候会休眠一段时间直到收到响应为止。
>[Kafka 网络层源码分析](https://generalthink.github.io/2019/04/09/kafka-producer-network-request/)

#### 自动发送源码分析

那么自动发送offset又是在哪里完成的呢？同样的仍然是在consumer.poll()中完成的,之前分析的在poll之前需要加入Group,也就是和GroupCoordinator通信，在这个过程中就会处理offset的自动提交。
>Kafka consumer加入group的分析可以查看这篇[文章](https://generalthink.github.io/2019/05/15/how-to-join-kafka-consumer-group/)

源码在ConsumerCoordinator.poll()方法中，核心源码如下：

```java
public boolean poll(final long timeoutMs) {
  final long startTime = time.milliseconds();
  long currentTime = startTime;
  long elapsed = 0L;

  // 1. 处理回调
  invokeCompletedOffsetCommitCallbacks();

  // 这里省略掉加入Group的源码

  // 2. 自动提交offset
  maybeAutoCommitOffsetsAsync(currentTime);
  return true;
}

public void maybeAutoCommitOffsetsAsync(long now) {
  
  // 1. 开启了自动提交并且当前时间大于等于下一次自动提交的截止时间
  if (autoCommitEnabled && now >= nextAutoCommitDeadline) {

      // 2. 更新截止时间
      this.nextAutoCommitDeadline = now + autoCommitIntervalMs;
      
      // 3. 获取所有已被消费的TopPartition的offset信息
      Map<TopicPartition, OffsetAndMetadata> allConsumedOffsets = subscriptions.allConsumed();

      // 4. 异步提交offset，源码参考上面的异步提交方式
      commitOffsetsAsync(allConsumedOffsets, new OffsetCommitCallback() {});
  }
}

```

也就是经过下面几个步骤就实现了自动提交了

1. 开启自动提交并且当前时间大于等于下一次自动提交的截止时间
>this.nextAutoCommitDeadline = time.milliseconds() + autoCommitIntervalMs;  
>nextAutoCommitDeadline截止时间等于new KafkaConsumer的系统时间加上自动提交间隔时间

2. 更新截止时间为现在的时间加上自动提交间隔时间

3. 获取所有已被消费的TopicPartiton的offset信息

4. 异步提交offset


第三步走获取所有已被消费的partition的offset信息，这里不得不说到SubscriptionState这个类，我们来关注下它重要的成员变量

```java
public class SubscriptionState {

  // 该consumer订阅的所有topics
  private final Set<String> subscription;

  // 该consumer所属的group中，所有consumer订阅的topic。该字段只对consumer leader有用
  private final Set<String> groupSubscription;

  // partition分配好之后，该字段记录每个partition的消费状态(包含reset策略以及上次消费的position等）
  private final Map<TopicPartition, TopicPartitionState> assignment;
```

所以通过SubscriptionState我们就可以获取已被消费的TopicPartition的offset信息,根据这个可以获取所有已消费的offset,然后发送请求commit offset.
