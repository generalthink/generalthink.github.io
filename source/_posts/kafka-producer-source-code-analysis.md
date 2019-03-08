---
title: 我花了一周读了Kafka Producer的源码
date: 2019-03-07 16:29:16
tags: Kafka
keywords: KafkaProducer,Kafka producer源码分析,Kafka 生产者源码分析,Kafaka producer 参数设置,Kafka partition分区策略
---

talk is easy,show me the code,先来看一段创建producer的代码

```java
public class KafkaProducerDemo {

  public static void main(String[] args) {

    KafkaProducer<String,String> producer = createProducer();

    //指定topic,key,value
    ProducerRecord<String,String> record = new ProducerRecord<>("test1","newkey1","newvalue1");

    //异步发送
    producer.send(record);
    producer.close();

    System.out.println("发送完成");

  }

  public static KafkaProducer<String,String> createProducer() {
    Properties props = new Properties();

    //bootstrap.servers 必须设置
    props.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, "192.168.239.131:9092");

    // key.serializer   必须设置
    props.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class.getName());

    // value.serializer  必须设置
    props.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, StringSerializer.class.getName());

    //client.id
    props.put(ProducerConfig.CLIENT_ID_CONFIG, "client-0");

    //retries
    props.put(ProducerConfig.RETRIES_CONFIG, 3);

    //acks
    props.put(ProducerConfig.ACKS_CONFIG, "all");

    //max.in.flight.requests.per.connection
    props.put(ProducerConfig.MAX_IN_FLIGHT_REQUESTS_PER_CONNECTION, 1);
    
    //linger.ms
    props.put(ProducerConfig.LINGER_MS_CONFIG, 100);

    //batch.size
    props.put(ProducerConfig.BATCH_SIZE_CONFIG, 10240);

    //buffer.memory
    props.put(ProducerConfig.BUFFER_MEMORY_CONFIG, 10240);

    return new KafkaProducer<>(props);
  }
}
```
<!--more-->

生产者的API使用还是比较简单,创建一个ProducerRecord对象(**这个对象包含目标主题和要发送的内容,当然还可以指定键以及分区**),然后调用send方法就把消息发送出去了。在发送ProducerRecord对象时，生产者要先把键和值对象序列化成字节数组，这样才能在网络上进行传输。
在深入源码之前，我先给出一张源码分析图给大家(其实应该在结尾的时候给出来),这样看着图再看源码跟容易些
![流程图](/images/introduction-kafka/kafka_producer_process.png)

简要说明:
1. `new KafkaProducer()`后创建一个后台线程KafkaThread(实际运行线程是Sender,KafkaThread是对Sender的封装)扫描RecordAccumulator中是否有消息

2. 调用`KafkaProducer.send()`发送消息，实际是将消息保存到RecordAccumulator中,实际上就是保存到一个Map中(`ConcurrentMap<TopicPartition, Deque<ProducerBatch>>`),这条消息会被记录到同一个记录批次(相同主题相同分区算同一个批次)里面,这个批次的所有消息会被发送到相同的主题和分区上

3. 后台的独立线程扫描到`RecordAccumulator`中有消息后，会将消息发送到kafka集群中(不是一有消息就发送，而是要看消息是否ready)

4. 如果发送成功(消息成功写入kafka),就返回一个`RecordMetaData`对象，它包换了主题和分区信息，以及记录在分区里的偏移量。

5. 如果写入失败，就会返回一个错误，生产者在收到错误之后会尝试重新发送消息(**如果允许的话,此时会将消息在保存到RecordAccumulator中**),几次之后如果还是失败就返回错误消息



### 源码分析

#### 后台线程的创建
```
KafkaClient client = new NetworkClient(...);
this.sender = new Sender(.,client,...);
String ioThreadName = "kafka-producer-network-thread" + " | " + clientId;
this.ioThread = new KafkaThread(ioThreadName, this.sender, true);
this.ioThread.start();
```
上面的代码就是构造`KafkaProducer`时核心逻辑,它会构造一个`KafkaClient`负责和broker通信,同时构造一个`Sender`并启动一个异步线程，这个线程会被命名为：**kafka-producer-network-thread|${clientId}**,如果你在创建producer的时候指定`client.id`的值为myclient,那么线程名称就是**kafka-producer-network-thread|myclient**

#### 发送消息(缓存消息)

```java
KafkaProducer<String,String> producer = createProducer();

//指定topic,key,value
ProducerRecord<String,String> record = new ProducerRecord<>("test1","newkey1","newvalue1");

//异步发送,可以设置回调函数
producer.send(record);
//同步发送
//producer.send(record).get();
```

发送消息有同步发送以及异步发送两种方式，我们一般不使用同步发送，毕竟太过于耗时，使用异步发送的时候可以指定回调函数，当消息发送完成的时候(成功或者失败)会通过回调通知生产者。

发送消息实际上是将消息缓存起来，核心代码如下：

```java
RecordAccumulator.RecordAppendResult result = accumulator.append(tp, timestamp, 
  serializedKey,serializedValue, headers, interceptCallback, remainingWaitMs);

```

`RecordAccumulator`的核心数据结构是`ConcurrentMap<TopicPartition, Deque<ProducerBatch>>`,会将相同主题相同Partition的数据放到一个Deque(双向队列)中,这也是我们之前提到的同一个记录批次里面的消息会发送到同一个主题和分区的意思。append()方法的核心源码如下：

```java
//从batchs(ConcurrentMap<TopicPartition, Deque<ProducerBatch>>)中
//根据主题分区获取对应的队列，如果没有则new ArrayDeque<>返回
Deque<ProducerBatch> dq = getOrCreateDeque(tp);

//计算同一个记录批次占用空间大小，batchSize根据batch.size参数决定
int size = Math.max(this.batchSize, AbstractRecords.estimateSizeInBytesUpperBound(
	maxUsableMagic, compression, key, value, headers));

//为同一个topic,partition分配buffer，如果同一个记录批次的内存不足，
//那么会阻塞maxTimeToBlock(max.block.ms参数)这么长时间
ByteBuffer buffer = free.allocate(size, maxTimeToBlock);
synchronized (dq) {
  //创建MemoryRecordBuilder,通过buffer初始化appendStream(DataOutputStream)属性
  MemoryRecordsBuilder recordsBuilder = recordsBuilder(buffer, maxUsableMagic);
  ProducerBatch batch = new ProducerBatch(tp, recordsBuilder, time.milliseconds());

  //将key,value写入到MemoryRecordsBuilder中的appendStream(DataOutputStream)中
  batch.tryAppend(timestamp, key, value, headers, callback, time.milliseconds());

  //将需要发送的消息放入到队列中
  dq.addLast(batch);
}

```

#### 发送消息到Kafka

上面已经将消息存储`RecordAccumulator`中去了,现在看看怎么发送消息。上面我们提到了创建KafkaProducer的时候会启动一个异步线程去从RecordAccumulator中取得消息然后发送到Kafka,发送消息的核心代码是`Sender.java`,它实现了Runnable接口并在后台一直运行处理发送请求并将消息发送到合适的节点，直到KafkaProducer被关闭

```java
/**
* The background thread that handles the sending of produce requests to the Kafka cluster. This thread makes metadata
* requests to renew its view of the cluster and then sends produce requests to the appropriate nodes.
*/
public class Sender implements Runnable {

  public void run() {

    // 一直运行直到kafkaProducer.close()方法被调用
    while (running) {
       run(time.milliseconds());
    }
    
    //从日志上看是开始处理KafkaProducer被关闭后的逻辑
    log.debug("Beginning shutdown of Kafka producer I/O thread, sending remaining records.");

    //当非强制关闭的时候，可能还仍然有请求并且accumulator中还仍然存在数据，此时我们需要将请求处理完成
    while (!forceClose && (this.accumulator.hasUndrained() || this.client.inFlightRequestCount() > 0)) {
       run(time.milliseconds());
    }
    if (forceClose) {
        //如果是强制关闭,且还有未发送完毕的消息，则取消发送并抛出一个异常new KafkaException("Producer is closed forcefully.")
        this.accumulator.abortIncompleteBatches();
    }
    ...
  }
}

```
KafkaProducer的关闭方法有2个,`close()`以及`close(long timeout,TimeUnit timUnit)`,其中timeout参数的意思是等待生产者完成任何待处理请求的最长时间，第一种方式的timeout为Long.MAX_VALUE毫秒,如果采用第二种方式关闭，当timeout=0的时候则表示强制关闭,直接关闭Sender(设置running=false)。

run(long)方法中我们先跳过对transactionManager的处理，查看发送消息的主要流程如下：

```java
//将记录批次转移到每个节点的生产请求列表中
long pollTimeout = sendProducerData(now);

//轮询进行消息发送
client.poll(pollTimeout, now);

```

首先查看sendProducerData()方法，它的核心逻辑在`sendProduceRequest()`方法(处于Sender.java)中

```java
for (ProducerBatch batch : batches) {
    TopicPartition tp = batch.topicPartition;

    //将ProducerBatch中MemoryRecordsBuilder转换为MemoryRecords(发送的数据就在这里面)
    MemoryRecords records = batch.records();
    produceRecordsByPartition.put(tp, records);
}

ProduceRequest.Builder requestBuilder = ProduceRequest.Builder.forMagic(minUsedMagic, acks, timeout,
        produceRecordsByPartition, transactionalId);

//消息发送完成时的回调
RequestCompletionHandler callback = new RequestCompletionHandler() {
    public void onComplete(ClientResponse response) {
        //处理响应消息
        handleProduceResponse(response, recordsByPartition, time.milliseconds());
    }
};

//根据参数构造ClientRequest,此时需要发送的消息在requestBuilder中
ClientRequest clientRequest = client.newClientRequest(nodeId, requestBuilder, now, acks != 0,
        requestTimeoutMs, callback);

//将clientRequest转换成Send对象(Send.java,包含了需要发送数据的buffer)，
//给KafkaChannel设置该对象，记住这里还没有发送数据
client.send(clientRequest, now);
```

上面的client.send()方法最终会定位到NetworkClient.doSend()方法，所有的请求(无论是producer发送消息的请求还是获取metadata的请求)都是通过该方法设置对应的Send对象。所支持的请求在ApiKeys.java中都有定义，这里面可以看到每个请求的request以及response对应的数据结构。


上面只是设置了发送消息所需要准备的内容，现在进入到发送消息的主流程，发送消息的核心代码在Selector.java的pollSelectionKeys()方法中，代码如下:

```java
/* if channel is ready write to any sockets that have space in their buffer and for which we have data */
if (channel.ready() && key.isWritable()) {
  //底层实际调用的是java8 GatheringByteChannel的write方法
  channel.write();
}
```
就这样，我们的消息就发送到了broker中了,发送流程分析完毕，这个是完美的情况，但是总会有发送失败的时候(消息过大或者没有可用的leader)，那么发送失败后重发又是在哪里完成的呢?还记得上面的回调函数吗？没错，就是在回调函数这里设置的，先来看下回调函数源码

```java
private void handleProduceResponse(ClientResponse response, Map<TopicPartition, ProducerBatch> batches, long now) {
  RequestHeader requestHeader = response.requestHeader();

  if (response.wasDisconnected()) {
    //如果是网络断开则构造Errors.NETWORK_EXCEPTION的响应
    for (ProducerBatch batch : batches.values())
        completeBatch(batch, new ProduceResponse.PartitionResponse(Errors.NETWORK_EXCEPTION), correlationId, now, 0L);

  } else if (response.versionMismatch() != null) {

   //如果是版本不匹配，则构造Errors.UNSUPPORTED_VERSION的响应
    for (ProducerBatch batch : batches.values())
        completeBatch(batch, new ProduceResponse.PartitionResponse(Errors.UNSUPPORTED_VERSION), correlationId, now, 0L);

  } else {
    
    if (response.hasResponse()) {
        //如果存在response就返回正常的response
           ...
        }
    } else {

        //如果acks=0，那么则构造Errors.NONE的响应，因为这种情况只需要发送不需要响应结果
        for (ProducerBatch batch : batches.values()) {
            completeBatch(batch, new ProduceResponse.PartitionResponse(Errors.NONE), correlationId, now, 0L);
        }
    }
  }
}
```

而在completeBatch方法中我们主要关注失败的逻辑处理，核心源码如下：


```java
private void completeBatch(ProducerBatch batch, ProduceResponse.PartitionResponse response, long correlationId,
                           long now, long throttleUntilTimeMs) {
  Errors error = response.error;

  //如果发送的消息太大，需要重新进行分割发送
  if (error == Errors.MESSAGE_TOO_LARGE && batch.recordCount > 1 &&
        (batch.magic() >= RecordBatch.MAGIC_VALUE_V2 || batch.isCompressed())) {

    this.accumulator.splitAndReenqueue(batch);
    this.accumulator.deallocate(batch);
    this.sensors.recordBatchSplit();

  } else if (error != Errors.NONE) {

    //发生了错误，如果此时可以retry(retry次数未达到限制以及产生异常是RetriableException)
    if (canRetry(batch, response)) {
        if (transactionManager == null) {
            //把需要重试的消息放入队列中，等待重试，实际就是调用deque.addFirst(batch)
            reenqueueBatch(batch, now);
        } 
    } 
}
```

Producer发送消息的流程已经分析完毕，现在回过头去看流程图会更加清晰。

更多关于Kafka协议的涉及可以参考这个[链接](https://cwiki.apache.org/confluence/display/KAFKA/A+Guide+To+The+Kafka+Protocol)

#### 分区算法

```java
List<PartitionInfo> partitions = cluster.partitionsForTopic(topic);
int numPartitions = partitions.size();
if (keyBytes == null) {
    //如果key为null,则使用Round Robin算法
    int nextValue = nextValue(topic);
    List<PartitionInfo> availablePartitions = cluster.availablePartitionsForTopic(topic);
    if (availablePartitions.size() > 0) {
        int part = Utils.toPositive(nextValue) % availablePartitions.size();
        return availablePartitions.get(part).partition();
    } else {
        // no partitions are available, give a non-available partition
        return Utils.toPositive(nextValue) % numPartitions;
    }
} else {
    // 根据key进行散列
    return Utils.toPositive(Utils.murmur2(keyBytes)) % numPartitions;
}
```
Kafka中对于分区的算法有两种情况
1. 如果键值为null,并且使用了默认的分区器，那么记录键随机地发送到主题内各个可用的分区上。分区器使用轮询(Round Robin)算法键消息均衡地分布到各个分区上。
2. 如果键不为空，并且使用了默认的分区器，那么Kafka会对键进行散列(使用Kafka自己的散列算法，即使升级Java版本，散列值也不会发生变化)，然后根据散列值把消息映射到特定的分区上。同一个键总是被映射到同一个分区上(如果分区数量发生了变化则不能保证)，映射的时候会使用主题所有的分区，而不仅仅是可用分区，所以如果写入数据分区是不可用的，那么就会发生错误，当然这种情况很少发生。


如果你想要实现自定义分区，那么只需要实现Partitioner接口即可。


### 生产者的配置参数

分析了KafkaProducer的源码之后，我们会发现很多参数是贯穿在整个消息发送流程，下面列出了一些KafkaProducer中用到的配置参数。

1. acks
  acks参数指定了必须要有多少个分区副本收到该消息，producer才会认为消息写入是成功的。有以下三个选项
 
	+ acks=0,生产者不需要等待服务器的响应，也就是说如果其中出现了问题，导致服务器没有收到消息，生产者就无从得知，消息也就丢失了，当时由于不需要等待响应，所以可以以网络能够支持的最大速度发送消息，从而达到很高的吞吐量。

	+ acks=1, 只需要集群的leader收到消息，生产者就会收到一个来自服务器的成功响应。如果消息无法到达leader，生产者会收到一个错误响应，此时producer会重发消息。不过如果一个没有收到消息的节点称为leader，消息还是会丢失。

	+ acks=all,当所有参与复制的节点全部收到消息的时候，生产者才会收到一个来自服务器的成功响应，最安全不过延迟比较高。

2. buffer.memory

  设置生产者内存缓冲区的大小，如果应用程序发送消息的速度超过生产者发送到服务器的速度，那么就会导致生产者空间不足，此时send()方法要么被阻塞，要么抛出异常。取决于如何设置max.block.ms，表示在抛出异常之前可以阻塞一段时间。

3. retries

  发送消息到服务器收到的错误可能是可以临时的错误(比如找不到leader),这种情况下根据该参数决定生产者重发消息的次数。注意：此时要根据重试次数以及是否是RetriableException来决定是否重试。

4. batch.size

  当有多个消息需要被发送到同一个分区的时候，生产者会把他们放到同一个批次里面(Deque),该参数指定了一个批次可以使用的内存大小，按照字节数计算，当批次被填满，批次里的所有消息会被发送出去。不过生产者并不一定会等到批次被填满才发送，半满甚至只包含一个消息的批次也有可能被发送。

5. linger.ms

  指定了生产者在发送批次之前等待更多消息加入批次的时间。KafkaProducer会在批次填满或linger.ms达到上限时把批次发送出去。把linger.ms设置成比0大的数，让生产者在发送批次之前等待一会儿，使更多的消息加入到这个批次，虽然这样会增加延迟，当时也会提升吞吐量。

6. max.block.ms

  指定了在调用send()方法或者partitionsFor()方法获取元数据时生产者的阻塞时间。当生产者的发送缓冲区已满，或者没有可用的元数据时，这些方法就会阻塞。在阻塞时间达到max.block.ms时,就会抛出new TimeoutException("Failed to allocate memory within the configured max blocking time " + maxTimeToBlockMs + " ms.");

7. client.id

  任意字符串，用来标识消息来源，我们的后台线程就会根据它来起名儿，线程名称是kafka-producer-network-thread|{client.id}

8. max.in.flight.requests.per.connection

  该参数指定了生产者在收到服务器响应之前可以发送多少个消息。它的值越高，就会占用越多的内存，不过也会提升吞吐量。把它设为1可以保证消息是按照发送的顺序写入服务器的，即便发生了重试。

9. timeout.ms、request.timeout.ms和metadata.fetch.timeout.ms

  request.timeout.ms指定了生产者在发送数据时等待服务器返回响应的时间，metadata.fetch.timeout.ms指定了生产者在获取元数据(比如目标分区的leader)时等待服务器返回响应的时间。如果等待响应超时，那么生产者要么重试发送数据，要么返回一个错误。timeout.ms指定了broker等待同步副本返回消息确认的时间，与asks的配置相匹配——如果在指定时间内没有收到同步副本的确认，那么broker就会返回一个错误。

10. max.request.size

  该参数用于控制生产者发送的请求大小。broker对可接收的消息最大值也有自己的限制(message.max.bytes),所以两边的配置最好可以匹配，避免生产者发送的消息被broker拒绝。

11. receive.buffer.bytes和send.buffer.bytes

  这两个参数分别制定了TCP socket接收和发送数据包的缓冲区大小(和broker通信还是通过socket)。如果他们被设置为-1，就使用操作系统的默认值。如果生产者或消费者与broker处于不同的数据中心，那么可以适当增大这些值，因为跨数据中心的网络一般都有比较高的延迟和比较低的带宽。
