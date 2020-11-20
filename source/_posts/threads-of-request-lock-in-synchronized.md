---
title: 那些去请求锁的线程都怎么样了?
date: 2020-11-10 22:06:51
tags: 多线程
---

不知道你有没有想过，那些去申请锁的线程都怎样了？有些可能申请到了锁，马上就能执行业务代码。但是如果有一个锁被很多个线程需要，那么这些线程是如何被处理的呢？

今天我们走进synchronized 重量级锁，看看那些没有申请到锁的线程都怎样了。

ps: 如果你不想看分析结果,可以拉到最后，末尾有一张总结图,一图胜千言

<!--more-->

之前文章分析过synchroinzed中锁的优化，但是如果存在大量竞争的情况下，那么最终还是都会变成重量级锁。所以我们这里开始直接分析重量级锁的代码。

### 申请锁
在ObjectMonitor::enter函数中，有很多判断和优化执行的逻辑，但是核心还是通过EnterI函数实际进入队列将将当前线程阻塞

```c++
void ObjectMonitor::EnterI(TRAPS) {
  Thread * const Self = THREAD;

  // CAS尝试将当前线程设置为持有锁的线程
  if (TryLock (Self) > 0) {
    assert(_succ != Self, "invariant");
    assert(_owner == Self, "invariant");
    assert(_Responsible != Self, "invariant");
    return;
  }

  // 通过自旋方式调用tryLock再次尝试，操作系统认为会有一些微妙影响
  if (TrySpin(Self) > 0) {
    assert(_owner == Self, "invariant");
    assert(_succ != Self, "invariant");
    assert(_Responsible != Self, "invariant");
    return;
  }
  ...

  // 将当前线程构建成ObjectWaiter
  ObjectWaiter node(Self);
  Self->_ParkEvent->reset();
  node._prev   = (ObjectWaiter *) 0xBAD;
  node.TState  = ObjectWaiter::TS_CXQ;


  ObjectWaiter * nxt;
  for (;;) {
    // 通过CAS方式将ObjectWaiter对象插入CXQ队列头部中
    node._next = nxt = _cxq;
    if (Atomic::cmpxchg(&node, &_cxq, nxt) == nxt) break;

    // 由于cxq改变，导致CAS失败，这里进行tryLock重试
    if (TryLock (Self) > 0) {
      assert(_succ != Self, "invariant");
      assert(_owner == Self, "invariant");
      assert(_Responsible != Self, "invariant");
      return;
    }
  }

  // 阻塞当前线程
  for (;;) {
    if (TryLock(Self) > 0) break;
    assert(_owner != Self, "invariant");

    // park self
    if (_Responsible == Self) {
      Self->_ParkEvent->park((jlong) recheckInterval);
      recheckInterval *= 8;
      if (recheckInterval > MAX_RECHECK_INTERVAL) {
        recheckInterval = MAX_RECHECK_INTERVAL;
      }
    } else {
      Self->_ParkEvent->park();
    }
    ...
    
    if (TryLock(Self) > 0) break;

    ++nWakeups;

    if (TrySpin(Self) > 0) break;

    ...
  }

  ...

  // Self已经获取到锁了，需要将它从CXQ或者EntryList中移除
  UnlinkAfterAcquire(Self, &node);

  ...

}
```

1. 在入队之前,会调用tryLock尝试通过CAS操作将_owner(当前ObjectMonitor对象锁持有的线程指针)字段设置为Self(指向当前执行的线程),如果设置成功，表示当前线程获得了锁，否则没有。

```c++
int ObjectMonitor::TryLock(Thread * Self) {
  void * own = _owner;
  if (own != NULL) return 0;
  if (Atomic::replace_if_null(Self, &_owner)) {
    return 1;
  }
  return -1;
}
```

2. 如果tryLock没有成功，又会再次调用tryLock(trySpin中调用了tryLock)去尝试获取锁，因为这样可以告诉操作系统我迫切需要这个资源，希望能尽量分配给我。不过这种亲和力并不是一定能得到保证的协议，只是一种积极的操作。

3. 通过 ObjectWaiter对象将当前线程包裹起来，入到 CXQ 队列的头部

4. 阻塞当前线程(通过pthread_cond_wait)

5. 当线程被唤醒而获取了锁，调用UnlinkAfterAcquire方法将ObjectWaiter从CXQ或者EntryList中移除


### 核心数据结构

ObjectMonitor对象中保存了 sychronized 阻塞的线程的队列，以及实现了不同的队列调度策略，因此我们有必须先来认识下这个对象的一些重要属性

```c++
class ObjectMonitor {

  // mark word
  volatile markOop _header;

  // 指向拥有线程或BasicLock的指针                 
  void * volatile _owner; 

  // monitor的先前所有者的线程ID
  volatile jlong _previous_owner_tid;

  // 重入次数，第一次为0
  volatile intptr_t _recursions;

  // 下一个被唤醒的线程
  Thread * volatile _succ;

  // 线程在进入或者重新进入时被阻塞的列表,由ObjectWaiter组成,相当于对线程的一个封装对象
  ObjectWaiter * volatile _EntryList;

  // CXQ队列存储的是enter的时候因为锁已经被别的线程阻塞而进不来的线程
  ObjectWaiter * volatile _cxq;

  // 处于wait状态(调用了wait())的线程，会被加入到waitSet
  ObjectWaiter * volatile _WaitSet;

  // 省略其他属性以及方法

}

class ObjectWaiter : public StackObj {
 public:
  enum TStates { TS_UNDEF, TS_READY, TS_RUN, TS_WAIT, TS_ENTER, TS_CXQ };

  // 后一个节点
  ObjectWaiter * volatile _next;

  // 前一个节点
  ObjectWaiter * volatile _prev;

  // 线程
  Thread*       _thread;
  // 线程状态
  volatile TStates TState;
 public:
  ObjectWaiter(Thread* thread);
};

```

看到ObjectWaiter中的_next和_prev你就会明白，这是使用了双向队列实现等待队列的的，但是**实际上我们上面的入队操作并没有形成双向列表，形成双向列表是在exit锁的时候。**


### wait

Java Object 类提供了一个基于 native 实现的 wait 和 notify 线程间通讯的方式,JDK中wait/notify/notifyAll全部是通过native实现的，当然到了JVM，它的实现还是在 `src/hotspot/share/runtime/objectMonitor.cpp` 中。

```c++
void ObjectMonitor::wait(jlong millis, bool interruptible, TRAPS) {
 
  Thread * const Self = THREAD;
  JavaThread *jt = (JavaThread *)THREAD;

  ...

  // 如果线程被中断，需要抛出异常
  if (interruptible && Thread::is_interrupted(Self, true) && !HAS_PENDING_EXCEPTION) {
    THROW(vmSymbols::java_lang_InterruptedException());
    return;
  }
  
  jt->set_current_waiting_monitor(this);

  // 构造 ObjectWaiter节点
  ObjectWaiter node(Self);
  node.TState = ObjectWaiter::TS_WAIT;

  ...

  // 将ObjectWaiter加入WaitSet的尾部
  AddWaiter(&node);

  // 让出锁
  exit(true, Self);                    
 
  ...

  // 调研park()，阻塞当前线程
  if (interruptible && (Thread::is_interrupted(THREAD, false) || HAS_PENDING_EXCEPTION)) {
        // Intentionally empty
  } else if (node._notified == 0) {
    if (millis <= 0) {
      Self->_ParkEvent->park();
    } else {
      ret = Self->_ParkEvent->park(millis);
    }
  }
  ...
}

// 将node插入双向列表_WaitSet的尾部
inline void ObjectMonitor::AddWaiter(ObjectWaiter* node) {
  if (_WaitSet == NULL) {
    _WaitSet = node;
    node->_prev = node;
    node->_next = node;
  } else {
    ObjectWaiter* head = _WaitSet;
    ObjectWaiter* tail = head->_prev;
    tail->_next = node;
    head->_prev = node;
    node->_next = head;
    node->_prev = tail;
  }
```

上面我把wait的主要方法逻辑列出来了，主要会执行以下步骤

1. 首先判断当前线程是否被中断，如果被中断了需要抛出InterruptedException
2. 如果没有被中断，则会使用当前线程构造ObjectWaiter节点，将其插入双向链表WaitSet的尾部
3. 调用exit,让出锁(让出锁的逻辑会在后面分析)
4. 调用park(实际上是调用pthread_cond_wait)阻塞当前线程

### notify

同样的notify的逻辑也是在ObjectMonitory.cpp中

```c++
void ObjectMonitor::notify(TRAPS) {
  CHECK_OWNER();

  // waitSet为空，直接返回
  if (_WaitSet == NULL) {
    TEVENT(Empty-Notify);
    return;
  }
  DTRACE_MONITOR_PROBE(notify, this, object(), THREAD);
  
  // 唤醒某个线程
  INotify(THREAD);

  OM_PERFDATA_OP(Notifications, inc(1));
}

```

在notify中首先会判断waitSet是否为空，如果为空，表示没有线程在等待，则直接返回。否则则调用INotify方法。
>notifyAll方法实际上是循环调用INotify

```c++
void ObjectMonitor::INotify(Thread * Self) {

  // notify之前需要获取一个锁，保证并发安全
  Thread::SpinAcquire(&_WaitSetLock, "WaitSet - notify");

  // 移除并返回WaitSet中的第一个元素,比如之前waitSet中是1 <--> 2 <--> 3,现在是返回1，然后waitSet变成 2<-->3
  ObjectWaiter * iterator = DequeueWaiter();
  if (iterator != NULL) {
   
    // Disposition - what might we do with iterator ?
    // a.  add it directly to the EntryList - either tail (policy == 1)
    //     or head (policy == 0).
    // b.  push it onto the front of the _cxq (policy == 2).
    // For now we use (b).

    // 设置线程状态
    iterator->TState = ObjectWaiter::TS_ENTER;

    iterator->_notified = 1;
    iterator->_notifier_tid = JFR_THREAD_ID(Self);

    ObjectWaiter * list = _EntryList;
    if (list != NULL) {
      assert(list->_prev == NULL, "invariant");
      assert(list->TState == ObjectWaiter::TS_ENTER, "invariant");
      assert(list != iterator, "invariant");
    }

    // prepend to cxq
    if (list == NULL) {
      iterator->_next = iterator->_prev = NULL;
      _EntryList = iterator;
    } else {
      iterator->TState = ObjectWaiter::TS_CXQ;
      for (;;) {
        // 将需要唤醒的node放到CXQ的头部
        ObjectWaiter * front = _cxq;
        iterator->_next = front;
        if (Atomic::cmpxchg(iterator, &_cxq, front) == front) {
          break;
        }
      }
    }

    iterator->wait_reenter_begin(this);
  }

  // notify执行完成之后释放waitSet锁,注意这里并不是释放线程持有的锁
  Thread::SpinRelease(&_WaitSetLock);
}

```

notify的逻辑比较简单，就是将WaitSet的头节点从队列中移除，如果EntryList为空，则将出队节点放入到EntryList中，如果EntryList不为空，则将节点插入到CXQ列表的头节点。


需要注意的是,**notify并没有释放锁，释放锁的逻辑是在exit中**

### exit

当一个线程获得对象锁成功之后，就可以执行自定义的同步代码块了。执行完成之后会执行到 ObjectMonitor 的 exit 函数中，释放当前对象锁，方便下一个线程来获取这个锁。

```c++
void ObjectMonitor::exit(bool not_suspended, TRAPS) {
  Thread * const Self = THREAD;
  if (THREAD != _owner) {
    // 锁的持有者是当前线程
    if (THREAD->is_lock_owned((address) _owner)) {
      assert(_recursions == 0, "invariant");
      _owner = THREAD;
      _recursions = 0;
    } else {
      assert(false, "Non-balanced monitor enter/exit! Likely JNI locking");
      return;
    }
  }

  // 重入次数减去1
  if (_recursions != 0) {
    _recursions--;        // this is simple recursive enter
    return;
  }

  for (;;) {
    ...

    w = _EntryList;
    // 如果entryList不为空，则将
    if (w != NULL) {
      assert(w->TState == ObjectWaiter::TS_ENTER, "invariant");
      // 执行unpark,让出锁
      ExitEpilog(Self, w);
      return;
    }

    w = _cxq;
    ...

    _EntryList = w;
    ObjectWaiter * q = NULL;
    ObjectWaiter * p;

    // 这里将_cxq或者说_EntryList从单向链表变成了一个双向链表
    for (p = w; p != NULL; p = p->_next) {
      guarantee(p->TState == ObjectWaiter::TS_CXQ, "Invariant");
      p->TState = ObjectWaiter::TS_ENTER;
      p->_prev = q;
      q = p;
    }
    w = _EntryList;
    if (w != NULL) {
      guarantee(w->TState == ObjectWaiter::TS_ENTER, "invariant");
      // 执行unpark,让出锁
      ExitEpilog(Self, w);
      return;
    }
    ...
  }
  ...
}

void ObjectMonitor::ExitEpilog(Thread * Self, ObjectWaiter * Wakee) {
  // Exit protocol:
  // 1. ST _succ = wakee
  // 2. membar #loadstore|#storestore;
  // 2. ST _owner = NULL
  // 3. unpark(wakee)

  _succ = Wakee->_thread;

  ParkEvent * Trigger = Wakee->_event;

  Wakee  = NULL;

  // Drop the lock
  OrderAccess::release_store(&_owner, (void*)NULL);
  OrderAccess::fence();

  ...
  
  // 释放锁
  Trigger->unpark();

}

```

exit的逻辑还是比较简单的
1. 如果当前是当前线程要让出锁，那么则查看其重入次数是否为0，不为0则将重入次数减去1，然后直接退出。

2. 如果EntryList不为空，则将EntryList的头元素中的线程唤醒

3. 将cxq指针赋值给EntryList，然后通过循环将cxq链表变成双向链表，然后调用ExitEpilog将CXQ链表的头结点唤醒(实际是通过pthread_cond_signal)

从这里之后,EntryList和CXQ就是同一个了，因为将CXQ赋值给了EntryList了。

**需要注意的是这里唤醒的线程会继续执行文章开头的EnterI方法，此时会将ObjectWaiter从EntryList或者CXQ中移除。**

### 实战演示

上面的源码均是基于JDK12,JDK8中的代码关于exit和notify都还有其他策略(选择哪个线程)，而从JDK9开始就只保留了默认策略了。

所以下面的Java代码的运行结果无论是在jdk8还是jdk12,得到的结果都是一样的。

```java
Object lock = new Object();

Thread t1 = new Thread(() -> {
  System.out.println("Thread 1 start!!!!!!");
  synchronized (lock) {
    try {
      lock.wait();
    } catch (Exception e) {
    }
    System.out.println("Thread 1 end!!!!!!");
  }
});
Thread t2 = new Thread(() -> {
  System.out.println("Thread 2 start!!!!!!");
  synchronized (lock) {
      try {
        lock.wait();
      } catch (Exception e) {
      }
      System.out.println("Thread 2 end!!!!!!");
  }
});
Thread t3 = new Thread(() -> {
  System.out.println("Thread 3 start!!!!!!");
  synchronized (lock) {
      try {
        lock.wait();
      } catch (Exception e) {
      }
      System.out.println("Thread 3 end!!!!!!");
  }
});
Thread t4 = new Thread(() -> {
  System.out.println("Thread 4 start!!!!!!");
  synchronized (lock) {
    try {
      System.in.read();
    } catch (Exception e) {
    }
    lock.notify();
    lock.notify();
    lock.notify();
    System.out.println("Thread 4 end!!!!!!");
  }
});
Thread t5 = new Thread(() -> {
  System.out.println("Thread 5 start!!!!!!");
  synchronized (lock) {
      System.out.println("Thread 5 end!!!!!!");
  }
});
Thread t6 = new Thread(() -> {
  System.out.println("Thread 6 start!!!!!!");
  synchronized (lock) {
      System.out.println("Thread 6 end!!!!!!");
  }
});
Thread t7 = new Thread(() -> {
  System.out.println("Thread 7 start!!!!!!");
  synchronized (lock) {
      System.out.println("Thread 7 end!!!!!!");
  }
});
t1.start();
sleep_1_second();

t2.start();
sleep_1_second();

t3.start();
sleep_1_second();

t4.start();
sleep_1_second();

t5.start();
sleep_1_second();

t6.start();
sleep_1_second();

t7.start();

```

上面的代码很简单，我们来分析一下。

线程1,2,3都调用了wait,所以会阻塞，然后WaitSet的链表结构如下：


![WaitSet:1-->2-->3](/images/macos-jvm/sync_waitset.png)

线程4获取了锁，在等待一个输入

线程5,6,7也在等待锁，所以他们也会把阻塞，所以CXQ链表结构如下：


![CXQ:7-->6-->5](/images/macos-jvm/sync_cxq.png)

当线程4输入任意内容，并回车结束后(调用了其中的3个notify方法，但还未释放锁)


![EntryList:1](/images/macos-jvm/sync_entrylist.png)


![CXQ: 3-->2-->7-->6-->5](/images/macos-jvm/sync_cxq_2.png)

线程4让出锁之后，由于EntryList不为空，所以会先唤醒EntryList中的线程1,然后接下来会唤醒CXQ队列中的线程(后面你可以认为CXQ就是EntryList)
所以最终线程执行顺序为 4 1 3 2 7 6 5,我们的输出结果也能验证我们的结论

```
Thread 1 start!!!!!!
Thread 2 start!!!!!!
Thread 3 start!!!!!!
Thread 4 start!!!!!!
Thread 5 start!!!!!!
Thread 6 start!!!!!!
Thread 7 start!!!!!!
think123
Thread 4 end!!!!!!
Thread 1 end!!!!!!
Thread 3 end!!!!!!
Thread 2 end!!!!!!
Thread 7 end!!!!!!
Thread 6 end!!!!!!
Thread 5 end!!!!!!
```

### 一图胜千言

![wait/notify/monitorexit](/images/macos-jvm/monitor_running.png)
