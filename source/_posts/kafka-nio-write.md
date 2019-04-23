---
title: 从Kafka到NIO
date: 2019-04-22 10:51:18
tags: [Kafka,NIO]
---

在谈NIO之前，简单回顾下内核态和用户态

内核空间是Linux内核运行的空间，而用户空间是用户程序的运行空间，为了保证内核安全，它们之间是隔离的，即使用户的程序崩溃了，内核也不受影响。
内核空间可以执行任意命令，调用系统的一切资源，用户空间只能执行简单运算，不能直接调用系统资源(I/O,进程资源,内存分配，外设，计时器，网络通信等)，必须通过系统接口(又称 system call)，才能向内核发出指令。

![内核图](/images/introduction-kafka/linux-structure.png)

<!--more-->

用户进程通过系统调用访问系统资源的时候，需要切换到内核态，而这对应一些特殊的堆栈和内存环境，必须在系统调用前建立好。而在系统调用结束后，cpu会从内核态切回到用户态，而堆栈又必须恢复成用户进程的上下文。而这种切换就会有大量的耗时。


### 进程缓冲区
一般程序在读取文件的时候先申请一块内存数组，称为buffer，然后每次调用read，读取设定字节长度的数据，写入buffer。(用较小的次数填满buffer)。之后的程序都是从buffer中获取数据，当buffer使用完后，在进行下一次调用，填充buffer。这里的buffer我们称为用户缓冲区，它的目的是为了减少频繁I/O操作而引起频繁的系统调用，从而降低操作系统在用户态与核心态切换所耗费的时间。


### 内核缓冲区
除了在进程中设计缓冲区，内核也有自己的缓冲区。

当一个用户进程要从磁盘读取数据时，内核一般不直接读磁盘，而是将内核缓冲区中的数据复制到进程缓冲区中。

但若是内核缓冲区中没有数据，内核会把对数据块的请求，加入到请求队列，然后把进程挂起，为其它进程提供服务。

等到数据已经读取到内核缓冲区时，把内核缓冲区中的数据读取到用户进程中，才会通知进程，当然不同的io模型，在调度和使用内核缓冲区的方式上有所不同。

你可以认为，read是把数据从内核缓冲区复制到进程缓冲区。write是把进程缓冲区复制到内核缓冲区。

当然，write并不一定导致内核的写动作，比如os可能会把内核缓冲区的数据积累到一定量后，再一次写入。这也就是为什么断电有时会导致数据丢失。


所以，我们进行IO操作的请求过程如下:用户进程发起请求(调用系统函数)，内核接收到请求后(进程会从用户态切换到内核态),从I/O设备中获取数据到内核buffer中，再将内核buffer中的数据copy到用户进程的地址空间，该用户进程获取到数据后再响应客户端。

### I/O复用模型

JavaNIO使用了I/O复用模型

![I/O复用](/images/introduction-kafka/io_multi.png)

从图中可以看出，我们阻塞在select调用，等待数据报套接字变为可读。当select返回套接字可读这一条件的时候，我们调用recvfrom把所读数据从内核缓冲区复制到应用进程缓冲区。
>那么内核态怎么判断I/O流可读可写？
>内核针对读缓冲区和写缓冲区来判断是否可读可写

而java从1.5开始就使用epoll代替了之前的select,它对select有所增强，比较有特点的是epoll支持水平触发(epoll默认)和边缘触发两种方式。
>epoll比select高效主要几种在两点,这个可以参考知乎的这个回答(https://www.zhihu.com/question/20122137/answer/146866418)
>1. 减少用户态和内核态之间的文件句柄拷贝
>2. 减少对可读可写文件句柄的遍历

epoll和NIO的操作方式对应图如下:

![epoll](/images/introduction-kafka/nio_compare_epoll.png)

1. epoll_ctl    注册事件
2. epoll_wait   轮询所有的socket
3. 处理对应的事件

epoll中比较有趣的是水平触发(LT)和边缘触发(ET)。

水平触发(条件触发)：读缓冲区只要不为空，就一直会触发读事件；写缓冲区只要不满(发送得速度比写得速度快)，就一直会触发写事件。这个比较符合编程习惯，也是epoll的缺省模式。

边缘触发(状态触发)：读缓冲区的状态，从空转为非空的时候，触发1次；写缓冲区的状态，从满转为非满的时候，触发1次。比如你发送一个大文件，把写缓存区塞满了，之后缓存区可以写了，就会发生一次从满到不满的切换。


通过分析，我们可以看出： 
对于LT模式，要避免"写的死循环"问题：写缓冲区为满的概率很小，也就是"写的条件"会一直满足，所以如果你注册了写事件，没有数据要写，但它会一直触发，所以在LT模式下，写完数据，一定要取消写事件。

对应ET模式，要避免"short read"问题:比如你收到100个字节，它触发1次，但你只读到了50个字节，剩下的50个字节不读，它也不会再次触发，此时这个socket就废了。因此在ET模式，一定要把"读缓冲区"的数据读完。


### 验证

代码太长了，我就只列出一段服务器端的主要代码,client端的比较简单，写法和server端也类似就不列出来了
```
Selector selector = Selector.open();

// 创建通道ServerSocketChannel
ServerSocketChannel serverSocketChannel = ServerSocketChannel.open();
// 将通道设置为非阻塞
serverSocketChannel.configureBlocking(false);

ServerSocket serverSocket = serverSocketChannel.socket();
serverSocket.bind(new InetSocketAddress(8989));

/**
* 将通道(Channel)注册到通道管理器(Selector)，并为该通道注册selectionKey.OP_ACCEPT事件
* 注册该事件后，当事件到达的时候，selector.select()会返回，
* 如果事件没有到达selector.select()会一直阻塞。
*/
serverSocketChannel.register(selector, SelectionKey.OP_ACCEPT);

// 循环处理
while (true) {
    // 当注册事件到达时，方法返回，否则该方法会一直阻塞
    selector.select();

    // 获取监听事件
    Set<SelectionKey> selectionKeys = selector.selectedKeys();
    Iterator<SelectionKey> iterator = selectionKeys.iterator();

    // 迭代处理
    while (iterator.hasNext()) {
        // 获取事件
        SelectionKey key = iterator.next();

        // 移除事件，避免重复处理
        iterator.remove();

        // 客户端请求连接事件，接受客户端连接就绪
       if (key.isAcceptable()) {
           ServerSocketChannel server = (ServerSocketChannel) key.channel();
           SocketChannel socketChannel = server.accept();
           socketChannel.configureBlocking(false);
           // 给通道设置写事件，客户端监听到写事件后，进行读取操作
           socketChannel.register(selector, SelectionKey.OP_WRITE);
        } else if(key.isWritable()) {
            System.out.println("write");
            handleWrite(key);
        }
    }
}

```

当client连接上server的时候就会发现server一直收到写事件,**write会一直打印**。所以使用条件触发的API 时，如果应用程序不需要写就不要关注socket可写的事件，否则就会无限次的立即返回一个write ready通知。大家常用的select就是属于条件触发这一类，长期关注socket写事件会出现CPU 100%的毛病。所以在使用Java的NIO编程的时候，在没有数据可以往外写的时候要取消写事件，在有数据往外写的时候再注册写事件。

取消写事件可以这样写 `selectionKey.interestOps(key.interestOps() & ~SelectionKey.OP_WRITE);`

### Kafka中如何处理的

在上一篇对Kafka网络层的分析中,我们知道了它是通过NIO和服务端进行通信的。其中在KafkaChannel的send()方法里面有这样一段代码:

```java
private boolean send(Send send) throws IOException {
  send.writeTo(transportLayer);
  if (send.completed())
    transportLayer.removeInterestOps(SelectionKey.OP_WRITE);

  return send.completed();
}

```
请注意这里的**transportLayer.removeInterestOps(SelectionKey.OP_WRITE)**，它移除了注册的OP_WRITE事件。

既然取消了，肯定会添加。在发送数据之前KafkaChannel的setSend()方法里面又注册了OP_WRITE事件
```java
public void setSend(Send send) {
  if (this.send != null)
      throw new IllegalStateException("Attempt to begin a send operation with prior send operation still in progress, connection id is " + id);
  this.send = send;
  this.transportLayer.addInterestOps(SelectionKey.OP_WRITE);
}
```

所以还是那句话: **在没有数据可以往外写的时候要取消写事件，在有数据往外写的时候再注册写事件。**

