---
title: Kafka consumer如何加入consumer group
date: 2019-05-15 15:00:16
tags: Kafka
---

consumer比producer要复杂许多,producer没有组的概念，也不需要关注offset,而consumer不一样,它有组织(consumer group)，有纪律(offset)。这些对consumer的要求就会很高，这篇文章就先从consumer如何加入consumer group说起。

GroupCoordinator是运行在服务器上的一个服务,负责consumer以及offset的管理。消费者客户端的ConsumerCoordinator负责与GroupCoordinator进行通信。Broker在启动的时候，都会启动一个GroupCoordinator服务。

### 如何找到对应的GroupCoordinator节点？

对于 consumer group 而言，是根据其 group.id 进行 hash 并通过一定的计算得到其具对应的 partition 值(计算方式如下)，该 partition leader 所在 Broker 即为该 Group 所对应的 GroupCoordinator，GroupCoordinator 会存储与该 group 相关的所有的 Meta 信息。

>__consumer_offsets 这个topic 是 Kafka 内部使用的一个 topic，专门用来存储 group 消费的情况，默认情况下有50个 partition，每个 partition 默认三个副本。
>partition计算方式：abs(GroupId.hashCode()) % NumPartitions(其中，NumPartitions 是 __consumer_offsets 的 partition 数，默认是50个)。
>比如，现在通过计算abs(GroupId.hashCode()) % NumPartitions的值为35,那么就找第35个partition的leader在哪个broker(假设在192.168.1.12),那么GroupCoordinator节点就在这个broker。

同时这个消费者所提交的消费位移信息也会发送给这个partition leader所对应的broker节点，因此这个节点不仅是GroupCoordinator而且还保存分区分配方案和组内消费者位移。

<!--more-->

### 消息消费流程

先来回顾下消费消息的流程
```java

// 创建消费者
KafkaConsumer<String, String> consumer = new KafkaConsumer<>(props);

// 订阅主题
consumer.subscribe(Collections.singletonList("consumerCodeTopic"))

// 从服务端拉取数据
ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(100));

```

上面的代码就是消费消息的主体流程,在创建KafkaConsumer的时候会先创建ConsumerCoordinator,由它来负责和GroupCoordinator通信

```java
 this.coordinator = new ConsumerCoordinator(...);
```

接着就开始订阅主题，订阅的方式有好几种，我在上一篇文章有提到过。订阅的时候会设置当前订阅类型为SubscriptionType.AUTO_TOPICS,默认存在五种类型
```java
public class SubscriptionState {
 private enum SubscriptionType {
    // 默认
    NONE,
    // subscribe方式 
    AUTO_TOPICS, 
    // pattern方式订阅
    AUTO_PATTERN,
    // assign方式
    USER_ASSIGNED
  }
}
```

订阅完成后，就可以从服务器拉取数据了，consumer没有后台线程默默的拉取数据，它的所有行为都集中在poll()方法中，KafkaConsumer也是线程不安全的，同时只能允许一个线程运行。
因此在poll的时候会进行判定，如果有多个线程同时使用一个KafkaConsumer则会抛出异常
```java
private void acquire() {
  long threadId = Thread.currentThread().getId();
  if (threadId != currentThread.get() && !currentThread.compareAndSet(NO_CURRENT_THREAD, threadId))
      throw new ConcurrentModificationException("KafkaConsumer is not safe for multi-threaded access");
  refcount.incrementAndGet();
}
```

### 和GroupCoordinator取得联系

一个 Consumer 实例消费数据的前提是能够加入一个 group 成功，并获取其要订阅的 tp（topic-partition）列表，因此首先要做的就是和GroupCoordinator建立连接，加入组织，因此我们先把目光集中在ConsumerCoordinator。poll()方法中首先会去更新metadata信息(获取 GroupCoordinator 的ip以及接口,并连接、 join-Group、sync-group, 期间 group 会进行 rebalance )

```java
private ConsumerRecords<K, V> poll(final long timeoutMs, final boolean includeMetadataInTimeout) {

...

  do {

    client.maybeTriggerWakeup();

    final long metadataEnd;
    if (includeMetadataInTimeout) {
        
        if (!updateAssignmentMetadataIfNeeded(remainingTimeAtLeastZero(timeoutMs, elapsedTime))) {
            return ConsumerRecords.empty();
        }
       ...
    } else {
        while (!updateAssignmentMetadataIfNeeded(Long.MAX_VALUE)) {
            log.warn("Still waiting for metadata");
        }
        metadataEnd = time.milliseconds();
    }

    final Map<TopicPartition, List<ConsumerRecord<K, V>>> records = pollForFetches(remainingTimeAtLeastZero(timeoutMs, elapsedTime));
    ...

  } while (elapsedTime < timeoutMs);

...

}

boolean updateAssignmentMetadataIfNeeded(final long timeoutMs) {
  final long startMs = time.milliseconds();
  if (!coordinator.poll(timeoutMs)) {
      return false;
  }

  return updateFetchPositions(remainingTimeAtLeastZero(timeoutMs, time.milliseconds() - startMs));
}
```

关于对ConsumerCoordinator的处理都集中在coordinator.poll()方法中。其主要逻辑如下:

```java
public boolean poll(final long timeoutMs) {
  // 如果是subscribe方式订阅的
  if (subscriptions.partitionsAutoAssigned()) {

  // 检查心跳线程运行是否正常，如果心跳线程失败则抛出异常，反之则更新poll调用时间
   pollHeartbeat(currentTime);

   if (coordinatorUnknown()) {
        // 确保ConsumeCoordinator创建完成，如果没有则向服务器发送请求开始创建一个
        if (!ensureCoordinatorReady(remainingTimeAtLeastZero(timeoutMs, elapsed))) {
            return false;
        }
    }

  // 判断是需要重新加入group,如果订阅的partition变化或者分配的partition变化
    if (rejoinNeededOrPending()) {
        //确保group是active，加入group，分配订阅的partition
        if (!ensureActiveGroup(remainingTimeAtLeastZero(timeoutMs, elapsed))) {
            return false;
        }
  }

  // 如果设置的是自动commit,如果定时达到则自动commit
  maybeAutoCommitOffsetsAsync(currentTime);
 }

```
poll方法中，具体实现可以分为四个步骤

1. 检测心跳线程运行是否正常(需要定时向GroupCoordinator发送心跳,在建立连接之后,建立连接之前不会做任何事情)

2. 通过subscribe()方法订阅topic,如果 coordinator 未知，就初始化 Consumer Coordinator(在 ensureCoordinatorReady() 中实现，主要的作用是发送 FindCoordinatorRequest 请求，并建立连接）

3. 判断是否需要重新加入group,如果订阅的partition变化或者分配的partition变化时，需要rejoin,通过ensureActiveGroup()发送join-group、sync-group请求，加入group并获取其assign的TopicPartition list。

4. 如果设置的是自动commit,并且达到了发送时限则自动commit offset


关于rejoin,下列几种情况会触发再均衡(reblance)操作

+ 新的消费者加入消费组(第一次进行消费也属于这种情况)
+ 消费者宕机下线(长时间未发送心跳包)
+ 消费者主动退出消费组，比如调用unsubscrible()方法取消对主题的订阅
+ 消费组对应的GroupCoorinator节点发生了变化
+ 消费组内所订阅的任一主题或者主题的分区数量发生了变化

>取消topic订阅，consumer心跳线程超时以及在Server端给定的时间未收到心跳请求，这三个都是触发的LEAVE_GROUP请求


这其中我会重点介绍下第二步中的ensureCoordinatorReady()方法和第三步中的ensureActiveGroup()方法。


#### ensureCoordinatorReady

这个方法主要作用就是选择一个连接数最少的broker(还未响应请求最少的broker)，**发送FindCoordinator请求，并建立对应的TCP连接。**

+ 方法调用流程是ensureCoordinatorReady() –> lookupCoordinator() –> sendGroupCoordinatorRequest()。
+ 如果client收到response,那么就与GroupCoordinator建立连接

```java
protected synchronized boolean ensureCoordinatorReady(final long timeoutMs) {
 
  while (coordinatorUnknown()) {
      // 找到GroupCoordinator，并建立连接
      final RequestFuture<Void> future = lookupCoordinator();
      client.poll(future, remainingTimeAtLeastZero(timeoutMs, elapsedTime));
     ...
  }
  return !coordinatorUnknown();
}

// 找到coordinator
protected synchronized RequestFuture<Void> lookupCoordinator() {
  if (findCoordinatorFuture == null) {
      // 找一个最少连接的节点,
      Node node = this.client.leastLoadedNode();
      if (node == null) {
          log.debug("No broker available to send FindCoordinator request");
          return RequestFuture.noBrokersAvailable();
      } else
          // 发送FindCoordinator request请求
          findCoordinatorFuture = sendFindCoordinatorRequest(node);
  }
  return findCoordinatorFuture;
}

// 发送FindCoordinator请求
private RequestFuture<Void> sendFindCoordinatorRequest(Node node) {
    log.debug("Sending FindCoordinator request to broker {}", node);
    FindCoordinatorRequest.Builder requestBuilder =
            new FindCoordinatorRequest.Builder(FindCoordinatorRequest.CoordinatorType.GROUP, this.groupId);
    // 发送请求，并将response转换为RequestFuture
    return client.send(node, requestBuilder)
                 .compose(new FindCoordinatorResponseHandler());
}

// 根据response返回的ip以及端口和GroupCoordinator建立连接
private class FindCoordinatorResponseHandler extends RequestFutureAdapter<ClientResponse, Void> {

  @Override
  public void onSuccess(ClientResponse resp, RequestFuture<Void> future) {
      log.debug("Received FindCoordinator response {}", resp);

      FindCoordinatorResponse findCoordinatorResponse = (FindCoordinatorResponse) resp.responseBody();
      Errors error = findCoordinatorResponse.error();
      if (error == Errors.NONE) {
          synchronized (AbstractCoordinator.this) {
              int coordinatorConnectionId = Integer.MAX_VALUE - findCoordinatorResponse.node().id();
              AbstractCoordinator.this.coordinator = new Node(
                      coordinatorConnectionId,
                      findCoordinatorResponse.node().host(),
                      findCoordinatorResponse.node().port());
              // 初始化连接
              client.tryConnect(coordinator);
              // 更新心跳时间
              heartbeat.resetTimeouts(time.milliseconds());
          }
          future.complete(null);
      } 
      ...
  }
```

上面代码主要作用就是往一个负载最小的节点发起FindCoordinator请求(client.send()方法底层使用的是java NIO,在上一篇文章中分析过),Kafka在走到这个请求后会根据group_id查找对应的GroupCoordinator节点，如果找到对应的则会返回其对应的node_id,host和port信息

>这里的GroupCoordinator节点的确定在文章开头提到过,是通过group.id和partitionCount来确定的

上面的代码我把一些无关紧要的精简过了,你可以更多的关注注释，我在关键代码上都添加了对应的注释,对于理解主要流程挺有帮助。


#### ensureActiveGroup()

现在已经知道了GroupCoordinator节点,并建立了连接。ensureActiveGroup()这个方法的主要作用是**向GroupCoordinator发送join-group、sync-group请求，获取assign的TopicPartition list。**
1. 调用过程是ensureActiveGroup() -> ensureCoordinatorReady() -> startHeartbeatThreadIfNeeded() -> joinGroupIfNeeded()
2. joinGroupIfNeeded()方法中最重要的方法是initiateJoinGroup(),它的的调用流程是 disableHeartbeatThread() -> sendJoinGroupRequest() -> JoinGroupResponseHandler::handle()
-> onJoinLeader(),onJoinFollower() -> sendSyncGroupRequest()

```java
boolean ensureActiveGroup(long timeoutMs, long startMs) {
   
  if (!ensureCoordinatorReady(timeoutMs)) {
      return false;
  }

  // 启动心跳线程
  startHeartbeatThreadIfNeeded();

  long joinStartMs = time.milliseconds();
  long joinTimeoutMs = remainingTimeAtLeastZero(timeoutMs, joinStartMs - startMs);
  return joinGroupIfNeeded(joinTimeoutMs, joinStartMs);
  }

```

心跳线程就是在这里启动的，但是并不一定马上发送心跳包，会在满足条件之后才会开始发送。后面最主要的逻辑就集中在joinGroupIfNeeded()方法，它的核心代码如下:

```java
boolean joinGroupIfNeeded(final long timeoutMs, final long startTimeMs) {
   while (rejoinNeededOrPending()) {
    if (!ensureCoordinatorReady(remainingTimeAtLeastZero(timeoutMs, elapsedTime))) {
        return false;
    }
    if (needsJoinPrepare) {
        // 如果是自动提交,则要开始提交offset以及在joingroup之前回调reblanceListener接口
        onJoinPrepare(generation.generationId, generation.memberId);
        needsJoinPrepare = false;
    }

    // 发送join group请求
    final RequestFuture<ByteBuffer> future = initiateJoinGroup();

    ...

    if (future.succeeded()) {
        
        ByteBuffer memberAssignment = future.value().duplicate();

        // 发送完成
        onJoinComplete(generation.generationId, generation.memberId, generation.protocol, memberAssignment);

        resetJoinGroupFuture();
        needsJoinPrepare = true;
    }
}

```
initiateJoinGroup()的核心代码如下:

```java
private synchronized RequestFuture<ByteBuffer> initiateJoinGroup() {
       
  if (joinFuture == null) {
   
   // 禁止心跳线程运行
    disableHeartbeatThread();

    // 设置当前状态为rebalance
    state = MemberState.REBALANCING;
    // 发送joinGroup请求
    joinFuture = sendJoinGroupRequest();
    joinFuture.addListener(new RequestFutureListener<ByteBuffer>() {
      @Override
      public void onSuccess(ByteBuffer value) {
          
        synchronized (AbstractCoordinator.this) {
            // 设置状态为stable状态    
            state = MemberState.STABLE;
            rejoinNeeded = false;

            if (heartbeatThread != null)
                // 允许心跳线程继续运行
                heartbeatThread.enable();
        }
      }
      @Override
      public void onFailure(RuntimeException e) {
          
          synchronized (AbstractCoordinator.this) {
              // 如果joinGroup失败，设置状态为unjoined
              state = MemberState.UNJOINED;
          }
      }
    });
  }
  return joinFuture;
}

```

可以看到在joinGroup之前会让心跳线程暂时停下来，此时会将ConsumerCoordinator的状态设置为rebalance状态，当joinGroup成功之后会将状态设置为stable状态，同时让之前停下来的心跳继续运行。

在发送joinGroupRequest之后,收到服务器的响应，会针对这个响应在做一些重要的事情

```java
 private class JoinGroupResponseHandler extends CoordinatorResponseHandler<JoinGroupResponse, ByteBuffer> {
  @Override
  public void handle(JoinGroupResponse joinResponse, RequestFuture<ByteBuffer> future) {
    Errors error = joinResponse.error();
    if (error == Errors.NONE) {
      synchronized (AbstractCoordinator.this) {
          if (state != MemberState.REBALANCING) {
            future.raise(new UnjoinedGroupException());
          } else {
            AbstractCoordinator.this.generation = new Generation(joinResponse.generationId(),
                    joinResponse.memberId(), joinResponse.groupProtocol());
            // 如果当前consumer是leader
            if (joinResponse.isLeader()) {
                onJoinLeader(joinResponse).chain(future);
            } else {
              // 是follower
                onJoinFollower().chain(future);
            }
          }
      }
    }
    ...
  }

```

1. 当 GroupCoordinator 接收到 consumer 的 join-group 请求后，由于此时这个 group 的 member 列表还是空(group 是新建的，每个 consumer 实例被称为这个 group 的一个 member)，第一个加入的 member 将被选为 leader，也就是说，对于一个新的 consumer group 而言，当第一个 consumer 实例加入后将会被选为 leader。如果后面leader挂了，会从其他member里面随机选择一个menmber成为新的leader

2. 如果GroupCoordinator接收到consumer发送join-group请求，会将所有menber列表以及分区策略返回其response
3. consumer在收到GroupCoordinator的response后，如果这个consumer是group的leader,那么这个consumer会负责为整个group 分配分区(默认是range策略,分区策略会在后续文章进行讲解),然后leader会发送SyncGroup(sendSyncGroupRequest()方法)请求，将分配信息发送给GroupCoordinator,作为follower的consumer则只是发送一个空列表。

4. GroupCoordinator 在接收到leader发送来的请求后，会将assign的结果返回给所有已经发送sync-group请求的consumer实例。并且group的状态会转变为stable.如果后续再收到 sync-group 请求，由于 group 的状态已经是 Stable，将会直接返回其分配结果。

sync-group发送请求核心代码如下

```java

//根据返回的assignmenet strategy name来决定采用何种策略进行分区分配
Map<String, ByteBuffer> groupAssignment = performAssignment(joinResponse.leaderId(), joinResponse.groupProtocol(),
        joinResponse.members());
SyncGroupRequest.Builder requestBuilder =
        new SyncGroupRequest.Builder(groupId, generation.generationId, generation.memberId, groupAssignment);
log.debug("Sending leader SyncGroup to coordinator {}: {}", this.coordinator, requestBuilder);
return sendSyncGroupRequest(requestBuilder);

```
这个阶段主要是讲分区分配方案同步给各个消费者，这个同步仍然是通过GroupCoordinator来转发的。

>分区策略并非由leader消费者来决定，而是各个消费者投票决定的,谁的票多就采用什么分区策略。这里的分区策略是通过partition.assignment.strategy参数设置的，可以设置多个。如果选举除了消费者不支持的策略，那么就会抛出异常IllegalArgumentException: Member does not support protocol

经过上面的步骤，一个consumer实例就已经加入group成功了，加入group成功后，将会触发ConsumerCoordinator 的 onJoinComplete() 方法，其作用就是：更新订阅的 tp 列表、更新其对应的 metadata 及触发注册的 listener。

然后消费者就进入正常工作状态,同时消费者也通过向GroupCoordinator发送心跳来维持它们与消费者的从属关系，已经它们对分区的所有权关系。只要以正常的间隔发送心跳，就被认为是活跃的，但是如果GroupCoordinator没有响应，那么就会发送LeaveGroup请求退出消费组。

