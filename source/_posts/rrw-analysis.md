---
title: ReentrantReadWriteLock源码分析
date: 2020-12-16 15:24:30
tags: [java,多线程]
---


针对读多写少的场景，Java提供了另外一个实现Lock接口的读写锁ReentrantReadWriteLock(RRW),之前分析过ReentrantLock是一个独占锁，同一时间只允许一个线程访问。

而 RRW 允许多个读线程同时访问，但不允许写线程和读线程、写线程和写线程同时访问。
读写锁内部维护了两个锁，一个是用于读操作的ReadLock，一个是用于写操作的 WriteLock。

读写锁遵守以下三条基本原则

1. 允许多个线程同时读共享变量；
2. 只允许一个线程写共享变量；
3. 如果一个写线程正在执行写操作，此时禁止读线程读共享变量。

<!--more-->

### 读写锁如何实现

RRW也是基于AQS实现的，它的自定义同步器(继承自AQS)需要在同步状态state上维护多个读线程和一个写线程的状态。RRW的做法是使用高低位来实现一个整形控制两种状态，一个int占4个字节，一个字节8位。所以高16位表示读，低16位表示写。


```java
abstract static class Sync extends AbstractQueuedSynchronizer {

  static final int SHARED_SHIFT   = 16;

  // 10000000000000000(65536)
  static final int SHARED_UNIT    = (1 << SHARED_SHIFT);

  // 65535
  static final int MAX_COUNT      = (1 << SHARED_SHIFT) - 1;

  //1111111111111111
  static final int EXCLUSIVE_MASK = (1 << SHARED_SHIFT) - 1;

  // 读锁(共享锁)的数量,只计算高16位的值
  static int sharedCount(int c)    { return c >>> SHARED_SHIFT; }

  // 写锁(独占锁)的数量
  static int exclusiveCount(int c) { return c & EXCLUSIVE_MASK; }
 }

```

### 获取读锁

当线程获取读锁时，首先判断同步状态低16位，如果存在写锁，则获取锁失败，进入CLH队列阻塞，反之，判断当前线程是否应该被阻塞，如果不应该阻塞则尝试 CAS 同步状态，获取成功更新同步锁为读状态。


```java
 protected final int tryAcquireShared(int unused) {
           
  Thread current = Thread.currentThread();
  int c = getState();
  // 如果当前已经有写锁了，则获取失败
  if (exclusiveCount(c) != 0 &&
      getExclusiveOwnerThread() != current)
      return -1;
  // 获取读锁数量
  int r = sharedCount(c);

  // 非公平锁实现中readerShouldBlock()返回true表示CLH队列中有正在排队的写锁
  // CAS设置读锁的状态值
  if (!readerShouldBlock() &&
      r < MAX_COUNT &&
      compareAndSetState(c, c + SHARED_UNIT)) {

      // 省略记录获取readLock次数的代码

      return 1;
  }

  // 针对上面失败的条件进行再次处理
  return fullTryAcquireShared(current);
}

final int fullTryAcquireShared(Thread current) {
  
  // 无线循环
  for (;;) {
    int c = getState();
    if (exclusiveCount(c) != 0) {
      // 如果不是当前线程持有写锁，则进入CLH队列阻塞
      if (getExclusiveOwnerThread() != current)
        return -1;
    } 

    // 如果reader应该被阻塞
    else if (readerShouldBlock()) {
        // Make sure we're not acquiring read lock reentrantly
        if (firstReader == current) {
            // assert firstReaderHoldCount > 0;
        } else {
            if (rh == null) {
                rh = cachedHoldCounter;
                if (rh == null || rh.tid != getThreadId(current)) {
                    rh = readHolds.get();
                    if (rh.count == 0)
                        readHolds.remove();
                }
            }
            // 当前线程没有持有读锁，即不存在锁重入情况。则进入CLH队列阻塞
            if (rh.count == 0)
                return -1;
        }
    }

    // 共享锁的如果超出了限制
    if (sharedCount(c) == MAX_COUNT)
        throw new Error("Maximum lock count exceeded");

    // CAS设置状态值
    if (compareAndSetState(c, c + SHARED_UNIT)) {
      
      // 省略记录readLock次数的代码

      return 1;
    }
  }

}
```

`SHARED_UNIT`的值是65536，也就是说，当第一次获取读锁的后，state的值就变成了65536。
**在公平锁的实现中当CLH队列中有排队的线程，`readerShouldBlock()`方法就会返回为true。非公平锁的实现中则是当CLH队列中存在等待获取写锁的线程就返回true**

还需要注意的是获取读锁的时候，如果当前线程已经持有写锁，是仍然能获取读锁成功的。后面会提到锁的降级，如果你对那里的代码有疑问，可以在回过头来看看这里申请锁的代码

### 释放读锁


```java
protected final boolean tryReleaseShared(int unused) {
           
  for (;;) {
    int c = getState();
    // 减去65536
    int nextc = c - SHARED_UNIT;
    // 只有当state的值变成0才会真正的释放锁
    if (compareAndSetState(c, nextc))
        return nextc == 0;
}
}
```

释放锁时，state的值需要减去65536，因为当第一次获取读锁后，state值变成了65536。

任何一个线程释放读锁的时候只有在`state==0`的时候才真正释放了锁，比如有100个线程获取了读锁，只有最后一个线程执行`tryReleaseShared`方法时才真正释放了锁，此时会唤醒CLH队列中的排队线程。


### 获取写锁

一个线程尝试获取写锁时，会先判断同步状态 state 是否为0。如果 state 等于 0，说明暂时没有其它线程获取锁；如果 state 不等于 0，则说明有其它线程获取了锁。

此时再判断state的低16位(w)是否为0，如果w为0，表示其他线程获取了读锁，此时进入CLH队列进行阻塞等待。

如果w不为0，则说明其他线程获取了写锁，此时需要判断获取了写锁的是不是当前线程，如果不是则进入CLH队列进行阻塞等待，如果获取了写锁的是当前线程，则判断当前线程获取写锁是否超过了最大次数，若超过，抛出异常。反之则更新同步状态。

```java
// 获取写锁
protected final boolean tryAcquire(int acquires) {
           
  Thread current = Thread.currentThread();
  int c = getState();
  int w = exclusiveCount(c);

  // 判断state是否为0
  if (c != 0) {
      // 获取锁失败
      if (w == 0 || current != getExclusiveOwnerThread())
          return false;

      // 判断当前线程获取写锁是否超出了最大次数65535
      if (w + exclusiveCount(acquires) > MAX_COUNT)
          throw new Error("Maximum lock count exceeded");
      
      // 锁重入
      setState(c + acquires);
      return true;
  }
  // 非公平锁实现中writerShouldBlock()永远返回为false
  // CAS修改state的值
  if (writerShouldBlock() ||
      !compareAndSetState(c, c + acquires))
      return false;

  // CAS成功后，设置当前线程为拥有独占锁的线程
  setExclusiveOwnerThread(current);
  return true;
}
```
在公平锁的实现中当CLH队列中存在排队的线程,那么`writerShouldBlock()`方法就会返回为true，此时获取写锁的线程就会被阻塞。


### 释放写锁

释放写锁的逻辑比较简单

```java
 protected final boolean tryRelease(int releases) {
  // 写锁是否被当前线程持有
  if (!isHeldExclusively())
      throw new IllegalMonitorStateException();
  
  int nextc = getState() - releases;
  boolean free = exclusiveCount(nextc) == 0;

  // 没有其他线程持有写锁
  if (free)
      setExclusiveOwnerThread(null);
  setState(nextc);
  return free;
}

```

### 锁的升级？


```java
// 准备读缓存
readLock.lock();
try {
  v = map.get(key);
  if(v == null) {
    writeLock.lock();
    try {
      if(map.get(key) != null) {
        return map.get(key);
      }

      // 更新缓存代码，省略
    } finally {
      writeLock.unlock();
    }
  }
} finally {
  readLock.unlock();
}

```
对于上面的代码，先是获取读锁，然后再升级为写锁，这样的行为叫做锁的升级。可惜RRW不支持，这样会导致写锁永久等待，最终导致线程被永久阻塞。所以**锁的升级是不允许的**。


### 锁的降级

虽然锁的升级不允许，但是锁的降级却是可以的。

```java
ReentrantReadWriteLock lock = new ReentrantReadWriteLock();

ReadLock readLock = lock.readLock();

WriteLock writeLock = lock.writeLock();

Map<String, String> dataMap = new HashMap();

public void processCacheData() {
  readLock.lock();

  if(!cacheValid()) {

    // 释放读锁，因为不允许
    readLock.unlock();

    writeLock.lock();

    try {
      if(!cacheValid()) {
          dataMap.put("key", "think123");
      }

      // 降级为读锁
      readLock.lock();
    } finally {
        writeLock.unlock();
    }
  }

  try {
    // 仍然持有读锁
    System.out.println(dataMap);
  } finally {
      readLock.unlock();
  }
}

public boolean cacheValid() {
    return !dataMap.isEmpty();
}

```


### RRW需要注意的问题

1. 在读取很多、写入很少的情况下，RRW 会使写入线程遭遇饥饿（Starvation）问题，也就是说写入线程会因迟迟无法竞争到锁而一直处于等待状态。

2. 写锁支持条件变量，读锁不支持。读锁调用newCondition() 会抛出UnsupportedOperationException 异常


### 推荐阅读

之前有写过AQS的实现，ReentrantLock的实现，可以参考我下面的文章

1. [AQS源码分析](https://generalthink.github.io/2020/11/20/sourcecode-of-AQS/)
2. [ReentrantLock分析](https://generalthink.github.io/2020/11/23/about-ReentrantLock-problems/)