---
title: AQS源码分析
date: 2020-11-20 14:21:24
tags: [Java, 多线程]
---

### 如何看待AQS?

看了Synchronized的实现方式之后，再来看JDK的AQS,感觉就比较简单了，它的行为有点像银行排队，银行有很多窗口处理很多业务，不同的窗口处理不同的业务，比如有个人业务，也有金融业务，也有公司业务等，每个窗口都有很多人排队。一般来讲当你在窗口处理业务的时候是不会有人来打扰你的，不一般的时候是什么时候呢？那就是资料分发窗口，谁来了都给你一张资料，一边看去吧。

而且当你去处理业务的时候，你可能有些资料忘记带了，然后你又要重新去取资料，取完之后回来继续排队。


所以说，代码源于生活，古人诚不欺我。

<!--more-->

### 带着问题看源码

1. AQS中是如何实现线程阻塞的？
2. AQS为什么支持多个Condition？
3. 线程唤醒的时候顺序是怎样的？

带着问题去看源码比一猛子扎进源码的海洋好，所以我建议你带着这几个问题去看AQS源码。

### CLH队列 

AQS是一个抽象类，很多类都是基于它来实现自己的特有的功能的，比如Lock,Semaphore,CountDownLatch等，我们都知道对于一个共享变量，为了线程安全，同一时刻肯定只能有一个线程线程可以写这个变量，也就是只有一个线程能拿到锁。那么那些没有拿到锁的线程是怎么被阻塞的呢？

AQS中维护了一个基于链表实现的FIFO队列(官方叫它CLH锁),那些没有获取到锁的线程会被封装后放入都在这个队列里面,我们来看下这个队列的定义

```java
static final class Node {

  // 共享模式
  static final Node SHARED = new Node();
  
  // 排它模式，默认
  static final Node EXCLUSIVE = null;

  // 当前线程的一个等待状态,默认值为0
   volatile int waitStatus;

  // 前一个节点
  volatile Node prev;

  // 后一个节点
  volatile Node next;

  // 入队到这个节点的线程
  volatile Thread thread;

  // 下一个等待condition的节点或者是特殊的节点值: SHARED
  Node nextWaiter;
}

```

这里共享模式和排它模式是什么意思呢？排它模式是表示同时只能有一个线程获取到锁执行，而共享模式表示可以同时有多个线程执行，比如Semaphore/CountDownLatch。

waitStatus除了默认值外还有三个值，分别代表不同的含义

1. CANCELLED = 1 : 表示当前节点已经不能在获取锁了，当前节点由于超时或者中断而进入该状态，进入该状态的节点状态不会再发生变化，同时后续还会从队列中移除。
2. SIGNAL = -1 : 表示当前节点需要去唤醒后继节点。 后继节点入队时，会将当前节点的状态更新为SIGNAL
3. CONDITION = -2 : 当前节点处于条件队列中，当其他线程调用了condition.signal()后，节点会从条件队列转义到同步队列中
4. PROPAGATE = -3 ： 当共享锁释放的时候，这个状态会被传递到其他后继节点


**其实原本的CLH队列是没有prev这个指针的，它存在的意义是那些取消锁获取的线程需要被移除掉。**

Synchronized的wait只能有一个条件队列(WaitSet)，而AQS则是支持多个条件队列,其中条件队列的关键数据结构如下:

```java

 public class ConditionObject implements Condition, java.io.Serializable {
  
  // condition queue中的头节点
  private transient Node firstWaiter;
  
  // condition queue中的尾节点
  private transient Node lastWaiter;

  /**
   * 创建一个新的condition
   */
  public ConditionObject() { }

  // 类似于Object的notify
  public final void signal() {
    ...
  }
  
  // 类似于Object的wait
  public final void await() throws InterruptedException {
    ...
  }

 }

```

AQS支持多个Condition的原因就是每个ConditionObject都对应一个队列,所以它可以支持多个Condition。


如果多个线程请求同一把锁，有的线程阻塞在请求锁上面，有的锁阻塞到等待某个条件成立。 那么当持有锁的线程让出锁的时候，哪个线程应该获取到锁呢？

当获取了锁的线程调用了signal()时，它又会不会从条件队列(condition queue)中出队一个node,加入到同步队列中呢？答案是会的。

如果我说AQS分析完毕，你们会不会打我？


哈哈哈，各位放下键盘，扶我起来，我还能在水一会儿。


AQS中除了这两个类之外，还有AQS本身还有几个重要的属性，它们共同构成了AQS这个框架类的基础结构

```java

  // wait queue(或者叫做同步队列)的头节点
  private transient volatile Node head;

  // wait queue(或者叫同步队列)的尾节点
  private transient volatile Node tail;

  // 资源
  private volatile int state;

```
看到了head和tail，是不是一个双向链表的含义跃然纸上了。

AQS 中的state变量，对不同的子类有不同的含义，该变量对 ReentrantLock 来说表示加锁状态，对Semaphore来说则是信号量的数量。总之它们都是基于AQS中的state来实现各自独有功能的。

这里的state我觉得更应该叫做资源，就类似于去银行排队时取的票。对于ReentrantLock来说，这个资源只有1个，对于Semaphore或者CountDownLatch来说资源可以有很多个。

### Unsafe讲解

unsafe是一个非常强大的类，通过它提供的API，我们可以操作内存，修改对象属性值，甚至内存屏障，加锁解锁都能通过它完成


![unsafe](/images/java/meituan_safe.png)


>图片来自于: https://tech.meituan.com/2019/02/14/talk-about-java-magic-class-unsafe.html


AQS中就使用了unsafe来实现线程的阻塞以及修改属性的值，这里把AQS中涉及到unsafe的方法列出来

```java

// 阻塞当前线程
public native void park(boolean isAbsolute, long time);

// 解锁指定的线程
public native void unpark(Object thread);

// 获取字段的偏移地址
public native long objectFieldOffset(Field f);

// CAS更新对象的值为x,当现在的值是expected
public final native boolean compareAndSwapObject(Object o, long offset,
                                                     Object expected,
                                                     Object x);
// CAS更新
public final native boolean compareAndSwapInt(Object o, long offset,
                                                  int expected,
                                                  int x);

```

park和unpark的底层实现其实还是之前我们讲解过的pthread,这里不再讲解，感兴趣的可以去看看之前关于synchronized的文章。

```cpp

UNSAFE_ENTRY(void, Unsafe_Park(JNIEnv *env, jobject unsafe, jboolean isAbsolute, jlong time))

...

 thread->parker()->park(isAbsolute != 0, time);
...

UNSAFE_END
```

修改对象的属性值，我们用一段代码来演示

```java
public class UnSafeDemo {

  public static void main(String[] args) throws Exception {

    Field theUnsafe = Unsafe.class.getDeclaredField("theUnsafe");
    theUnsafe.setAccessible(true);
    Unsafe unsafe = (Unsafe) theUnsafe.get(null);

    Student student = new Student();
    student.setAge(18);

    long ageOffset = unsafe.objectFieldOffset(Student.class.getDeclaredField("age"));

    // cas修改student的age为20
    unsafe.compareAndSwapInt(student, ageOffset, 18, 20);
    System.out.println(student.getAge());
  }

}


class Student {
  int age;

  public int getAge() {
      return age;
  }

  public void setAge(int age) {
      this.age = age;
  }
}

```

上面代码的输出结果为 `20`,我们通过首先通过objectFieldOffset获取到age属性的内存偏移地址，然后通过compareAndSwapInt将age属性的值修改为20。

> 比如对于一维数组a[10],基地址是0x000000,那么a[1]的内存地址就是0x000001 = 0x000000(基地址) + 1(偏移量)


### 源码分析

其他类如果要想将AQS作为基类以实现自己的功能，只需要根据需求实现以下方法就行了

```java
  // 尝试获取锁
  protected boolean tryAcquire(int arg) {
    throw new UnsupportedOperationException();
  }

  // 尝试释放锁
  protected boolean tryRelease(int arg) {
    throw new UnsupportedOperationException();
  }

  // 尝试获取共享锁
  protected int tryAcquireShared(int arg) {
    throw new UnsupportedOperationException();
  }

  // 尝试释放共享锁
  protected boolean tryReleaseShared(int arg) {
    throw new UnsupportedOperationException();
  }

  // 锁是否被独占
  protected boolean isHeldExclusively() {
    throw new UnsupportedOperationException();
  }

```

AQS中获取锁的方法是在`acquire()`中，释放锁的方法是`release()`，如果我们想要实现自定义的锁的时候，只需要根据自己的需求实现对应的tryXXX方法就行了，为了简化分析，我们只分析独占锁的源码。


#### 获取锁

```java
public final void acquire(int arg) {
  if (!tryAcquire(arg) &&
    acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
    selfInterrupt();
}
```

我们假设tryAcquire返回false(不关心子类实现),也就是现在锁已经被其他线程占有了。现在又有线程来获取锁肯定是会获取失败的，所以失败的线程会被封装成Node插入到队列中

```java
private Node addWaiter(Node mode) {
  
  // 将nextWaiter设置为EXCLUSIVE，表示节点正以独占模式等待
  Node node = new Node(Thread.currentThread(), mode);

  // Try the fast path of enq; backup to full enq on failure
  Node pred = tail;
  if (pred != null) {
      node.prev = pred;
      // CAS将tail节点更新为node
      if (compareAndSetTail(pred, node)) {
          pred.next = node;
          return node;
      }
  }

  // 如果执行到这里就存在两种情况
  // 1. pred == null,表示是第一次插入，当前同步队列中没有其他线程
  // 2. 表明有其他线程修改了tail的值,导致CAS修改tail失败
  enq(node);
  return node;
}


// CAS更新tail节点
private Node enq(final Node node) {
  for (;;) {
      Node t = tail;

      // 处理第一次插入的情况，这里head = new Node()
      if (t == null) {
          if (compareAndSetHead(new Node()))
              tail = head;
      } else {

          // 处理竞争激烈的情况或者第一次插入后未建立链路关系的情况
          node.prev = t;
          if (compareAndSetTail(t, node)) {
              t.next = node;
              return t;
          }
      }
  }
}

```

上面逻辑很简单，就是构造一个排它锁模式的node，插入队列中(尾插入)。

但是你尤其需要注意到enq中，当节点第一次插入的时候，head的值是new Node(),它是一个虚拟节点，这个节点本身没有可运行的线程。

> 在链表中，使用这种虚拟节点很正常，有时候更加有利于操作。

enq执行完成后就形成了这样的链路关系: 

![enq](/images/java/aqs-enq.png)

acquireQueued的源码如下

```java

final boolean acquireQueued(final Node node, int arg) {
  boolean failed = true;
  try {
    boolean interrupted = false;
    for (;;) {
      final Node p = node.predecessor();

      // 1. 如果前一个节点是头节点，则尝试再次获取锁(机会更大)
      if (p == head && tryAcquire(arg)) {
          setHead(node);
          p.next = null; // help GC
          failed = false;
          return interrupted;
      }

      // 2. 获取锁失败，需要检查并更新节点状态
      if (shouldParkAfterFailedAcquire(p, node) &&
          parkAndCheckInterrupt())
          interrupted = true;
    }
  } finally {
      // 3. 如果线程被中断，需要将节点从队列中移除
      if (failed)
          cancelAcquire(node);
  }
}
```
逻辑也比较简单明了，在线程被阻塞之前，如果前一个节点是头节点(head),那么在尝试去获取一次锁，如果成功了就直接返回。

如果还是失败的话，就更新下节点状态，然后将线程阻塞

```java
private static boolean shouldParkAfterFailedAcquire(Node pred, Node node) {
  int ws = pred.waitStatus;

  // 前置节点已经是SIGNAL了，可以放心的park了
  if (ws == Node.SIGNAL)
      return true;
  if (ws > 0) {
      // 如果前置节点状态是CANCELLED,需要将节点移除
      do {
          node.prev = pred = pred.prev;
      } while (pred.waitStatus > 0);
      pred.next = node;
  } else {
      // 这就是我们之前说的在入队的时候，会通过CAS将前置节点的状态设置为SIGNAL
      compareAndSetWaitStatus(pred, ws, Node.SIGNAL);
  }
  return false;
}

```

线程阻塞的代码更简单

```java
private final boolean parkAndCheckInterrupt() {

  // 调用了unsafe的park阻塞当前线程
  LockSupport.park(this);
  return Thread.interrupted();
}

```
至此，获取锁的源码就解析完成了。

从上面的代码，我们可以知道 `head --> next --> next --> tail` 这样的数据结构组成了同步队列，等待获取锁的线程会被封装成node插入到队列中去。


#### 释放锁

release()中主要调用了unparkSuccessor()方法来唤醒后继节点，源码如下

```java

// 传入的node是head节点
private void unparkSuccessor(Node node) {
  
  // 1. 获取结点状态
  int ws = node.waitStatus;

  // 2. 使用CAS更新waitStatus为默认值0
  if (ws < 0)
      compareAndSetWaitStatus(node, ws, 0);

  // 3. 通过循环找到后继节点中状态不为CANCELLED的节点
  Node s = node.next;
  if (s == null || s.waitStatus > 0) {
      s = null;
      for (Node t = tail; t != null && t != node; t = t.prev)
          if (t.waitStatus <= 0)
              s = t;
  }
  // 4. 调用unsafe解锁线程
  if (s != null)
      LockSupport.unpark(s.thread);
}

```

之前提到过head节点是虚拟节点，所以会调用`unsafe.park`去解锁head.next节点

#### 条件变量(Condition)

Java 语言内置的管程(synchronized)里只有一个条件变量，而我们的AQS是支持多个条件变量的，Java中定义了一个接口，叫做Condition,AQS的内部类ConditionObject实现了这个接口。

```java
public interface Condition {

 // 使得当前线程被阻塞，直到收到信号(signal)或者线程被中断
 void await() throws InterruptedException;

 // 唤醒一个等待线程
 void signal();
 
 // 唤醒所有的等待线程
 void signalAll();

 // 省略其他方法
}

```

Condition中线程等待和通知需要调用`await()`、`signal()`、`signalAll()`，它们的语义和 `wait()`、`notify()`、`notifyAll()` 是相同的。但是不一样的是，Condition里只能使用前面的 `await()`、`signal()`、`signalAll()`，而后面的 `wait()`、`notify()`、`notifyAll()` 只有在 synchronized 实现的管程里才能使用。

#### await

```java
public final void await() throws InterruptedException {
    if (Thread.interrupted())
        throw new InterruptedException();

    // 将当前线程构造成一个waiter节点，waitStatue的值为-2(CONDITION),将它插入到条件队列中
    Node node = addConditionWaiter();

    // 释放持有的锁
    int savedState = fullyRelease(node);
    int interruptMode = 0;

    // 如果不在同步队列中，则阻塞当前线程
    while (!isOnSyncQueue(node)) {
        LockSupport.park(this);
        if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
            break;
    }

    // 在同步队列中，有可能被唤醒了，需要去重新获取锁。并处理中断逻辑
    if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
        interruptMode = REINTERRUPT;
    if (node.nextWaiter != null) // clean up if cancelled
        unlinkCancelledWaiters();
    if (interruptMode != 0)
        reportInterruptAfterWait(interruptMode);
}


 private Node addConditionWaiter() {
  Node t = lastWaiter;
  
  // 如果lastWaiter不是出于condition状态，则需要移除掉
  if (t != null && t.waitStatus != Node.CONDITION) {
      unlinkCancelledWaiters();
      t = lastWaiter;
  }

  // 将当前线程构建为一个condition node
  Node node = new Node(Thread.currentThread(), Node.CONDITION);
  if (t == null)
      firstWaiter = node;
  else
      t.nextWaiter = node;

  // 修改lastWaiter的指向
  lastWaiter = node;
  return node;
}

```

await()方法的逻辑也比较简单，其执行的主要流程如下

1. 如果当前线程已经被中断则抛出异常，未中断则执行步骤2
2. 将当前线程构造为condition node，并插入到条件队列中
3. 如果condition node不在同步队列中(有可能被唤醒后，移出条件队列了)，则调用unsafe.park阻塞当前线程

通过以上代码分析，await之后条件队列会形成下面这样的数据结构

```
firstWaiter --> nextWaiter --> nextWaiter --> lastWaiter
```
> 这里的firsetWaiter并不是虚拟节点，当只有一个线程在条件队列中时，firstWaiter == node == lastWaiter

#### signal

```java

public final void signal() {
  if (!isHeldExclusively())
      throw new IllegalMonitorStateException();

  // 获取第一个等待节点，进行唤醒
  Node first = firstWaiter;
  if (first != null)
      doSignal(first);
}

private void doSignal(Node first) {
  do {
      // 将节点从条件队列中移除
      if ( (firstWaiter = first.nextWaiter) == null)
          lastWaiter = null;
      first.nextWaiter = null;
  } while (!transferForSignal(first) &&
           (first = firstWaiter) != null);
}

// 将节点从条件队列移动到同步队列
final boolean transferForSignal(Node node) {
 
  // CAS修改waitStatus失败，就证明node处于CANCELLED状态
  if (!compareAndSetWaitStatus(node, Node.CONDITION, 0))
      return false;

  // enq的源码上面上面获取锁有提到过，就是将节点插入同步队列并返回前置节点
  Node p = enq(node);
  int ws = p.waitStatus;
  // 如果前置节点状态是CANCELLED或者修改waitStatus状态为SIGNAL失败，那么需要唤醒刚插入的节点
  if (ws > 0 || !compareAndSetWaitStatus(p, ws, Node.SIGNAL))
      LockSupport.unpark(node.thread);
  return true;
}

```

其实signal的逻辑也是很简单的，不知道为什么写到signal的时候，我就想起了我当时写synchronized的时候，一样的索然无味，无欲无求。我想可能是没人给我点赞。

说回signal，它的主要逻辑如下

1. 将第一个等待节点从等待队列中移除
2. 修改第一个等待节点的waitStatus为0
3. 将节点入队到同步队列,并返回前一个节点
4. 判断前一个节点是否处于CANCELLED或者waitStatus能否正常修改，如果不能则解锁刚入队的节点

你需要注意的是，signal并没有释放自己获得的锁。释放锁的操作仍然是通过release的。


### 总结

通过源码的分析，我相信你一定可以回答上之前的三个问题。也让我们知道了AQS和Synchronized其实内在实现方式是很类似的。

AQS中有同步队列和条件队列，signal的时候是将节点从条件队列转移到同步队列。
Synchronized中有EntryList和WaitSet。notify的时候将线程从WaitSet移动到EntryList中。

同样的，看源码我个人更喜欢沿着一条线去看，而不是大而全的看，这样更容易把控整个脉络。
