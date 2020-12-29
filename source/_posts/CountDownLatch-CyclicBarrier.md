---
title: CountDownLatch和CyclicBarrier,我该怎么选?
date: 2020-12-28 14:42:19
tags: [java,多线程]
---

### CountDownLatch和CyclicBarrier区别

CountDownLatch 和 CyclicBarrier 是 Java 并发包提供的两个非常易用的线程同步工具类，把它们放在一起介绍是因为它们之间有点像，又很不同。

CountDownLatch 主要用来解决一个线程等待多个线程的场景，可以类比旅游团团长要等待所有的游客到齐才能去下一个景点。

CyclicBarrier 是一组线程之间互相等待，只要有一个线程没有完成，其他线程都要等待,更像是你和你老婆不离不弃。

对于CountDownLatch来说，重点是那**一个线程**, 是它在等待，而另外那N个线程在把“某个事情”做完之后可以继续等待，可以终止。

而对于CyclicBarrier来说，重点是那**一组(N个)**线程，他们之间任何一个没有完成，所有的线程都必须等待。

除此之外 CountDownLatch 的计数器是不能循环利用的，也就是说一旦计数器减到 0，再有线程调用 await()，该线程会直接通过。
但CyclicBarrier 的计数器是可以循环利用的，而且具备自动重置的功能，一旦计数器减到 0 会自动重置到你设置的初始值。除此之外，CyclicBarrier 还可以设置回调函数，可以说是功能丰富。

<!--more-->

### CountDownLatch使用

我们假设旅游团有3个游客，团长要等到游客都到齐了之后才能出发去下一个景点

```java
CountDownLatch latch = new CountDownLatch(3);
ExecutorService executor = Executors.newFixedThreadPool(3);

IntStream.rangeClosed(1, 3).forEach(i -> {
    executor.execute(() -> {
        System.out.println("游客" + i + "到了集合地点");
        latch.countDown();
    });
});

latch.await();

System.out.println("所有人员都已经到齐了，出发去下个景点");

```

首先创建了一个 CountDownLatch，计数器的初始值等于 3，之后每当一个团员到达就对计数器执行减 1操作(latch.countDown()实现)。在主线程中，我们通过调用 latch.await() 来实现对计数器等于 0 的等待。

### CountDownLatch源码分析

CountDownLatch是通过AQS来实现的，这里的计数器的值实际上是AQS中State的值,也就是我们的state的值会被初始化为我们传入的值。

当我们调用coutnDown的时候实际上是减去state的值(计数器减1)

```java
// CountDownLatch中代码
public void countDown() {
  sync.releaseShared(1);
}

private static final class Sync extends AbstractQueuedSynchronizer {

  // countDown会调用这个方法
  protected boolean tryReleaseShared(int releases) {
    for (;;) {
        int c = getState();
        if (c == 0)
            return false;
        // 计数器的值减1
        int nextc = c-1;
        if (compareAndSetState(c, nextc))
            return nextc == 0;
    }
}

}

// AQS中的代码
public final boolean releaseShared(int arg) {
  // 当state等于0的时候，去唤醒阻塞的线程
  if (tryReleaseShared(arg)) {
      doReleaseShared();
      return true;
  }
  return false;
}

```

调用await的时候，会判断当前state的值是否等于0，如果等于0,就代表其他线程已经执行完成了，可以接着往下执行。否则就阻塞当前线程。

```java
// CountDownLatch中代码
public void await() throws InterruptedException {
  sync.acquireSharedInterruptibly(1);
}

// await会调用这个方法
protected int tryAcquireShared(int acquires) {
  return (getState() == 0) ? 1 : -1;
}

// AQS中的代码
public final void acquireSharedInterruptibly(int arg)
        throws InterruptedException {

  if (Thread.interrupted())
      throw new InterruptedException();
  
  if (tryAcquireShared(arg) < 0)
      // 阻塞线程
      doAcquireSharedInterruptibly(arg);
}
```

至此逻辑就很清晰了。 当我们调用countDown()方法的时候，会将AQS中的state的值减去1,当state值变为0的时候会唤醒CLH队列中阻塞的线程。当我们调用await()方法的时候，会判断state的值是否等于0，如果等于0则继续往下执行。如果不等于0则线程被阻塞，等待被唤醒(countDown()方法中会唤醒)。


### CyclicBarrier使用

周末的时候，我和我老婆一起去吃烧烤，用CyclicBarrier来描述是这样的

```java

// 执行回调的线程池，很重要的点
ExecutorService executor = Executors.newFixedThreadPool(1);

// 指定计数器的值和回调函数
CyclicBarrier barrier = new CyclicBarrier(2, () -> {
    executor.execute(() -> System.out.println("到齐了，出发吃烧烤"));
});

new Thread(() -> {
  try {
    TimeUnit.SECONDS.sleep(2);
    System.out.println("半个小时后，think123媳妇儿准备好了");
    barrier.await();
  } catch (InterruptedException e) {
    e.printStackTrace();
  } catch (BrokenBarrierException e) {
    e.printStackTrace();
  }
}).start();

new Thread(() -> {
  System.out.println("think123准备好了");
  try {
    barrier.await();
  } catch (InterruptedException e) {
    e.printStackTrace();
  } catch (BrokenBarrierException e) {
    e.printStackTrace();
  }
}).start();

```

### CyclicBarrier源码分析

```java
public class CyclicBarrier {

  private static class Generation {
      boolean broken = false;
  }

  private final ReentrantLock lock = new ReentrantLock();
  
  private final Condition trip = lock.newCondition();
 
  private final int parties;
 
  private final Runnable barrierCommand;

  private Generation generation = new Generation();

  private int count;
}
```

上面展示了CyclicBarrier的重要属性,它的属性名称还是挺有趣的。
为了实现一组线程相互等待，使用到了lock和condition，而parties则是表明一组线程的个数(计数器)，count表示当前有多少个线程还未执行完成。 barrierCommand表示当所有线程都就绪时，需要回调的函数。而generation是为了实现计数器循环利用，你可以理解为版本。

接下来我们看看await方法是如何实现的，下面的代码我只保留了核心代码逻辑

```java

public int await() {
  return dowait(false, 0L);
}
private int dowait(boolean timed, long nanos) {

  final ReentrantLock lock = this.lock;

  // 加锁，独占锁
  lock.lock();
 
  final Generation g = generation;

  //省略部分代码
  
  // 还有多少线程未执行完成
  int index = --count;

  // 所有线程都执行完成
  if (index == 0) {
    
    // 执行barrierCommand回调函数
    final Runnable command = barrierCommand;

    // 注意这里调用的是Runnable的run方法，而不是thread.start()
    if (command != null)
        command.run();

    ranAction = true;
    // 重置计数器，并唤醒所有正在等待的线程
    nextGeneration();
    return 0;
      
  }

  // 只保留核心代码

  for (;;) {
      // 阻塞线程
      if (!timed)
          trip.await();
      else if (nanos > 0L)
          nanos = trip.awaitNanos(nanos);
      if (g != generation)
          return index;
  }
  lock.unlock();

}

private void nextGeneration() {
  // 唤醒所有正在休假的线程
  trip.signalAll();
  // set up next generation
  count = parties;
  generation = new Generation();
}

```

await的逻辑很简单，主要就是判断当前线程是否是最后一个执行完成的线程，如果是最后一个，则需要执行回调函数，然后唤醒其他所有被阻塞的线程并重置计数器。
如果不是最后一个执行完的，则阻塞当前线程。


尤其需要注意的CyclicBarrier的回调函数执行在一个回合里最后执行await()的线程上，而且是同步调用回调函数，调用完之后，才会开始第二回合。
所以回调函数如果不另开一线程异步执行，就起不到性能优化的作用了。


### 写到最后

你告诉我，要是你你怎么选？

如果觉得好看，记得三连哟。