---
title: Kafka Producer消息收发设计
date: 2019-04-26 10:53:25
tags: [Kafka,NIO]
keywords : Kafka发送数据,Kafka接收数据,NIO
---

前几篇文章分析了Kafka的发送流程以及NIO的使用方式，但是还是留下了不少坑，这里就对剩下的问题做一个总结。

### 收到的数据为什么要缓存起来?

Kafka中Selector读取从远端回来的数据的时候会先把收到的数据缓存起来
```java
private void attemptRead(SelectionKey key, KafkaChannel channel) throws IOException {
  //if channel is ready and has bytes to read from socket or buffer, and has no
  //previous receive(s) already staged or otherwise in progress then read from it
  if (channel.ready() && (key.isReadable() || channel.hasBytesBuffered()) && !hasStagedReceive(channel)
      && !explicitlyMutedChannels.contains(channel)) {
      NetworkReceive networkReceive;
      while ((networkReceive = channel.read()) != null) {
          madeReadProgressLastPoll = true;
          addToStagedReceives(channel, networkReceive);
      }
  }
}
```
在NetworkClient中，往下传的是一个完整的ClientRequest，进到Selector，暂存到channel中的，也是一个完整的Send对象(1个数据包)。但这个Send对象，交由底层的channel.write(Bytebuffer b)的时候，并不一定一次可以完全发送，可能要调用多次write，才能把一个Send对象完全发出去。这是因为write是非阻塞的，不是等到完全发出去，才会返回。

```java
Send send = channel.write();
if (send != null) {
    this.completedSends.add(send);
    this.sensors.recordBytesSent(channel.id(), send.size());
}
```

这里如果返回send==null就表示没有发送完毕，需要等到下一次Selector.poll再次进行发送。所以当下次发送的时候如果Channel里面的Send只发送了部分，那么此次这个node就不会处于ready状态，就不会从RecordAccumulator取出要往这个node发的数据,等到Send对象发送完毕之后，这个node才会处于ready状态，就又可以取出数据进行处理了。

同样，在接收的时候，channel.read(Bytebuffer b)，一个response也可能要read多次，才能完全接收。所以就有了上面的while循环代码。

### 如何确定消息接收完成?

从上面知道，底层数据的通信，是在每一个channel上面，2个源源不断的byte流，一个send流，一个receive流。 
send的时候，还好说，发送之前知道一个完整的消息的大小。
但是当我们接收消息response的时候，这个信息可能是不完整的(剩余的数据要晚些才能获得)，也可能包含不止一条消息。那么我们是怎么判断消息发送完毕的呢？
对于消息的读取我们必须考虑消息结尾是如何表示的,标识消息结尾通常有以下几种方式：

1. 固定的消息大小。
2. 将消息的长度作为消息的前缀。
3. 用一个特殊的符号来标识消息的结束。

很明显第一种和第三种方式不是很合适，因此Kafka采用了第二种方式来确定要发送消息的大小。在消息头部放入了4个字节来确定消息的大小。

```java
//接收消息，前4个字节表示消息的大小
public class NetworkReceive implements Receive {
  private final String source;

  //确定消息size
  private final ByteBuffer size;

  private final int maxSize;

  //整个消息response的buffer
  private ByteBuffer buffer;  

  public NetworkReceive(String source) {
      this.source = source;

      //分配4字节的头部
      this.size = ByteBuffer.allocate(4);
      this.buffer = null;
      this.maxSize = UNLIMITED;
  }
}

//消息发送,前4个字节表示消息大小
public class NetworkSend extends ByteBufferSend {

  public NetworkSend(String destination, ByteBuffer buffer) {
      super(destination, sizeDelimit(buffer));
  }

  private static ByteBuffer[] sizeDelimit(ByteBuffer buffer) {
      return new ByteBuffer[] {sizeBuffer(buffer.remaining()), buffer};
  }

  private static ByteBuffer sizeBuffer(int size) {
    //4个字节表示消息大小
    ByteBuffer sizeBuffer = ByteBuffer.allocate(4);
    sizeBuffer.putInt(size);
    sizeBuffer.rewind();
    return sizeBuffer;
  }

}
```

### OP_WRITE何时就绪?

上一篇文章虽然讲了epoll的原理，但是我相信还是有人觉得很迷惘，这里换个简单的说法再说下OP_WRITE事件。
OP_WRITE事件的就绪条件并不是发生在调用channel的write方法之后，也不是发生在调用channel.register(selector,SelectionKey.OP_WRITE)后,而是在当底层缓冲区有空闲空间的情况下。因为写缓冲区在绝大部分时候都是有空闲空间的，所以如果你注册了写事件，这会使得写事件一直处于写就绪，选择处理现场就会一直占用着CPU资源。所以，只有当你确实有数据要写时再注册写操作，并在写完以后马上取消注册。

### max.in.flight.requests.per.connection

这个参数指定了生产者在收到服务器响应之前可以发送多少个消息，找Kafka Producer中对应有一个类InFlightRequests,表示在天上飞的请求,也就是请求发出去了response还没有回来的请求数,这个参数也是判断节点是否ready的关键因素。只有ready的节点数据才能从Accumulator中取出来进行发送。

