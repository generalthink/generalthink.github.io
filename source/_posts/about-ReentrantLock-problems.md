---
title: ReentrantLock的这几个问题，你都知道吗？
date: 2020-11-23 10:21:05
tags: [多线程, java]
---

#### 公平锁和非公平锁的区别？

之前分析AQS的时候，了解到AQS依赖于内部的两个FIFO队列来完成同步状态的管理，当线程获取锁失败的时候，会将当前线程以及等待状态等信息构造成Node对象并将其加入同步队列中，同时会阻塞当前线程。当释放锁的时候，会将首节点的next节点唤醒(head节点是虚拟节点)，使其再次尝试获取锁。

同样的，如果线程因为某个条件不满足，而进行等待，则会将线程阻塞，同时将线程加入到等待队列中。当其他线程进行唤醒的时候，则会将等待队列中的线程出队加入到同步队列中使其再次获得执行权。

按照我们的分析，无论是同步队列还是等待队列都是FIFO,看起来就很公平呀？为什么ReentrankLock还分公平锁和不公平锁呢？


<!--more-->

还是直接看源码吧，看看它是怎么做的？


首先看看锁的创建

```java
// 默认是不公平锁 
public ReentrantLock() {
  sync = new NonfairSync();
}

// true表示公平锁，false表示不公平锁
public ReentrantLock(boolean fair) {
  sync = fair ? new FairSync() : new NonfairSync();
}
```


可以看到对应不同的锁，只是代表他们内部的Sync变量不同而已。

其中NonfairSync和FairSync两个类是Sync的子类，Sync又继承自AbstractQueuedSynchronizer

![公平锁和非公平锁继承关系](/images/java/reentranklock-sync-structure.png)


当我们使用ReentrantLock加锁的时候实际上调用的是sync.lock()方法,也就是说，我们需要看看他们加锁的时候有什么不同之处？


![lock的区别](/images/java/lock-difference.png)

可以看到在lock方法内部,非公平锁会先直接通过CAS修改state变量的值，如果修改成功则表示获取到了锁，而公平锁则是直接调用AQS的acquire方法来获取锁。

也就是说有可能当其他线程释放锁的时候，非公平锁能率先修改state的值成功，从而获取到锁。这样就比其他等待的线程率先获取到锁了，这就是不公平。


之前也有提到过，子类会根据自己的需求以实现tryAcquire方法，同样的非公平锁和公平锁的实现也实现了这个方法，我们可以来看看，两个的实现有什么不同


![区别](/images/java/lock-fair-unfair-difference.png)

可以看到公平锁比非公平锁的实现多了一个判断条件(!hasQueuedPredecessors()),我们来看看这个方法的实现

```java
public final boolean hasQueuedPredecessors() {
  Node t = tail;
  Node h = head;
  Node s;
  return h != t &&
      ((s = h.next) == null || s.thread != Thread.currentThread());
}
```

这个方法很简单，它的意思是如果当前线程之前有排队的线程，则返回true；如果当前线程位于队列的开头或队列为空，则返回false。

也就是锁公平锁在获取锁的时候会判断队列中是否已经有排队的线程，如果有则进行阻塞，如果没有则去通过CAS申请锁。

这就实现了公平锁，先来的先获取到锁，后来的后获取到锁。


所以我们可以总结下公平锁和非公平锁实现上的两点区别:

1. 非公平锁在调用lock()方法后，首先会通过CAS抢占锁，如果恰巧这个时候锁没有被占用，则获取锁成功
2. 非公平锁在CAS失败后，和公平锁一样会调用tryAcquire()方法，在tryAcquire()方法中，如果发现锁被释放了(state=0),非公平锁会直接CAS进行抢占，而公平锁会判断同步队列中是否有线程处于等待状态，如果有则不去抢占，而是排队获取。

这就是两者将细微的区别，如果这非公平锁两次CAS都失败了，那么会和公平锁一样，乖乖的在同步队列中排队。

相对而言，非公平锁的吞吐量更大，但是让获取锁的时间变得不确定，可能会导致同步队列中的线程长期处于饥饿状态。


### ReentrantLock靠什么保证可见性？

synchronized 之所以能够保证[可见性](https://generalthink.github.io/2020/06/02/start-of-concurrent-programming/),是因为有一条happens-before原则，那Java SDK 里面 ReentrantLock 靠什么保证可见性呢？

它是利用了 volatile 相关的 Happens-Before 规则。AQS内部有一个 volatile 的成员变量 state，当获取锁的时候，会读写state 的值；解锁的时候，也会读写 state 的值。
> 对一个volatile变量的写操作happens-before 于后面对这个变量的读操作。这里的happens-before是时间上的先后顺序

其实JVM在很多实现上都是有规范的，

这样说起来挺抽象的，我们直接去看JVM中对volatile是否有特殊的处理，在`src/hotspot/share/interpreter/bytecodeinterpreter.cpp`中，我们找到getfield和getstatic字节码执行的位置
> 现在这个执行器基本不再使用了，基本都会使用模板解释器，但是模板解释器的代码基本都是汇编，而我们只是想要快速了解其原理，所以可以看这个，对模板解释器感兴趣的可以去看templateTable_x86.cpp::getfield查看相关细节

```cpp
...
CASE(_getfield):
CASE(_getstatic):
{
   ...

   ConstantPoolCacheEntry* cache;
   ...
   if (cache->is_volatile()) {
     if (support_IRIW_for_not_multiple_copy_atomic_cpu) {
       OrderAccess::fence();
     }
     ...
   }
  ...
}
```

可以看到在访问对象字段的时候，会判断它是不是volatile的，如果是，且当前CPU平台支持多核atomic操作(现在大多数CPU都支持)，就调用`OrderAccess::fence()`。
> JDK中的Unsafe也提供了内存屏障的方法，在JVM层面也是通过OrderAccess实现


接下来来看下Linux x86下的实现是怎样的(src/hotspot/os_cpu/linux_x86/orderAccess_linux_x86.cpp)

```cpp
inline void OrderAccess::fence() {
// always use locked addl since mfence is sometimes expensive
#ifdef AMD64
  __asm__ volatile ("lock; addl $0,0(%%rsp)" : : : "cc", "memory");
#else
  __asm__ volatile ("lock; addl $0,0(%%esp)" : : : "cc", "memory");
#endif
  compiler_barrier();
}
```

指令中的"addl $0,0(%%esp)"(把ESP寄存器的值加0)是一个空操作，采用这个空操作而不是空操作指令nop是因为IA32手册规定lock前缀不允许配合nop指令使用，所以才采用加0这个空操作。

而lock有如下作用

1. lock锁定的时候，如果操作某个数据，那么其他CPU核不能同时操作
2. lock 锁定的指令，不能上下文随意排序执行，必须按照程序上下顺序执行
3. 在 lock 锁定操作完毕之后，如果某个数据被修改了，那么需要立即告诉其他 CPU 这个值被修改了，是它们的缓存数据立即失效，需要重新到内存获取


关于lock的实现有两种，一种是锁总线，一种是锁缓存。锁缓存就涉及到CPU Cache,缓存行以及MESI了，所以这里就不展开了，有兴趣的童鞋咱们可以私下交流下。