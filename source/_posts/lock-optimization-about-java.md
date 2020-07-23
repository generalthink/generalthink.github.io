title: java中锁的优化
date: 2018-07-19 21:22:46
tags: 多线程
categories: [java,jvm]
description: 多线程编码中,我们常用到synchronized关键字,都知道它是加锁关键字，也或多或少听说过轻量级锁，重量级锁，偏向锁这些关键字，可是我们知道这些具体指的是什么吗？为了我们更优的使用，JVM团队在背后做了多少努力?
keywords: synchronized,jvm,偏向锁,轻量级锁,重量级锁
---

### 一切都始于线程(Thread)

线程是比进程更加轻量级的调度执行单位,线程的引入可以把一个进程的资源分配和执行调服分开，各个线程之间共享进程资源又可以独立调度，主流的操作系统都提供了线程实现，Java提供了在不同硬件和不同操作系统平台上对线程操作的统一处理。

一般操作系统实现线程主要有3种方式：
1. 使用内核线程(直接由操作系统内核支持)实现,但是一般都是去使用内核线程的一种高级接口——轻量级进程
2. 使用用户线程(除了内核线程)实现
3. 使用用户线程加轻量级进程混合实现。

而java中的线程就是映射到一条轻量级进程中的,所以如果要阻塞或者唤醒一个线程都需要需要操作系统的支持,因为要从用户态转换到核心态中需要耗费很多的处理器时间，所以才有说法说synchronized是java中的一个重量级的操作。

### synchronized你对我的代码做了什么?

我们都知道Synchronized 在修饰同步代码块时，是由 monitorenter 和 monitorexit 指令来实现同步的。进入 monitorenter 指令后，线程将持有 Monitor 对象，退出 monitorenter 指令后，线程将释放该 Monitor 对象。

> 使用javap -v xxx.class 可以看到字节文件中的monitorenter和monitorexit指令

而我们一般可以在代码块中指明需要加锁的对象或者在根据修饰的是实例方法还是类方法(static方法)来获取对应的对应实例或者Class对象来进行加锁。

>当 Synchronized 修饰同步方法时，并不会出现 monitorenter 和 monitorexit 指令，而是出现了一个 ACC_SYNCHRONIZED 标志。这是因为 JVM 使用了 ACC_SYNCHRONIZED 访问标志来区分一个方法是否是同步方法。当方法调用时，调用指令将会检查该方法是否被设置 ACC_SYNCHRONIZED 访问标志。如果设置了该标志，执行线程将先持有 Monitor 对象，然后再执行方法。在该方法运行期间，其它线程将无法获取到该 Mointor 对象，当方法执行完成后，再释放该 Monitor 对象。


JVM中的synchronized是基于管程实现的,你可以认为管程就是信号量,synchronized实现加锁其操作系统底层实现方式是Pthread(基于信号量)。信号量我在之前操作系统的文章有提到过,不太清楚的可以再去看看。

信号量实际上是一个整形变量,比如 lock 初始值为0,加锁就将lock的值加1,释放锁就将lock的值减去1,加锁以及释放锁的过程由操作系统保证原子性。

synchronized也不外如是,它会在加锁之前先去获取锁，如果获取到了就把目标锁对象的计数器加1,释放锁就减去1，当计数器的值为0的时候就释放锁,monitorexit这个指令会在同步代码执行完毕的时候将计数器减去1。如果没有获取到锁，那么就要阻塞等待，直到对象锁被其他线程释放，在重新去竞争。

Java中每个对象实例都会有一个Monitor,其对应C++的ObjectMonitor.hpp

```c++
ObjectMonitor() {
  _header       = NULL;
  _count        = 0;
  _waiters      = 0,
  // 递归数量,和可重入锁有关
  _recursions   = 0;
  _object       = NULL;
  // 标识拥有该monitor的线程
  _owner        = NULL;
  // 处于wait状态的线程，会被加入到_WaitSet
  _WaitSet      = NULL;
  // 自旋锁
  _WaitSetLock  = 0 ;
  _Responsible  = NULL ;
  _succ         = NULL ;
  // contention queue(保存竞争线程的队列)
  _cxq          = NULL ;
  FreeNext      = NULL ;
  // 处于等待锁block状态的线程，会被加入到该列表
  _EntryList    = NULL ;
  // 自旋次数
  _SpinFreq     = 0 ;
  _SpinClock    = 0 ;
  OwnerIsThread = 0 ;
  _previous_owner_tid = 0;
}
```

![](/images/java/synchronized-process.png)

当多个线程同时访问一段同步代码时,多个竞争线程会先被存放到contention queue(cxq)中,cxq上的线程最终会被放置到EntryList,处于blocking状态的线程都会被加入到EntryList列表。
接下来当线程获取到对象的monitor时，monitor 是依靠底层操作系统的 Mutex Lock 来实现互斥的(实际就是信号量),线程阻塞操作由操作系统完成(在Linxu下通 过pthread_mutex_lock函数。当线程申请到Mutex成功，则持有该Mutex,其他线程将无法获取到Mutex,竞争失败的线程会再次进入contention queue被挂起。

如果线程调用了wait()方法,就会释放当前持有的Mutex,此时线程就会被放置到_WaitSet集合中，等待下一次被唤醒。如果当前线程顺利执行完方法，也会释放Mutex。
而处于_WaitSet集中的线程被唤醒后都会加入到_EntryList集合中。

> 关于contention queue说明可以在objectMonitor.cpp看到,mutex实现可以查看 mutex.cpp


### 锁的优化

为了提升性能，JDK1.6 引入了偏向锁、轻量级锁、重量级锁概念，来减少锁竞争带来的上下文切换，而正是新增的 Java 对象头实现了锁升级功能。

当 Java 对象被 Synchronized 关键字修饰成为同步锁后，围绕这个锁的一系列升级操作都将和 Java 对象头有关。

#### Java对象头

对象头分为两部分，第一部分存储hash码(hashcode),GC分代年龄等,这部分数据的长度在32位系统和64位系统分别为32位和64位,官方称其为“Mark Word",它们是实现偏向锁和轻量级锁的关键。另一部分用于存储指向方法区对象类型数据的指针，如果是数组对象的话，还会有一个额外的部分用于存储数组长度。

对象头信息与对象自定义的数据无关的额外存储成本，考虑虚拟机的空间效率，Mark Word被设计成一个非固定的的数据结构以便在极小的空间存储更多的信息，它会根据对象的状态复用自己的存储空间。

那么从哪里可以看到这些信息呢？有问题当然看源码，从github上jdk的源码[源码](https://github.com/openjdk/jdk/blob/jdk8-b120/hotspot/src/share/vm/oops/markOop.hpp#L38)




整理下，这里在内存中的布局如下：

![32位系统mark word](/images/mark_word.png)
上图表明了MarkWord在不同状态下的存储内容，当然上面这是32位系统的，如果是64位基本上是大同小异，


锁升级功能主要依赖于 Mark Word 中的锁标志位和释放偏向锁标志位，Synchronized 同步锁就是从偏向锁开始的，随着竞争越来越激烈，偏向锁升级到轻量级锁，最终升级到重量级锁。


monitorenter执行入口在InterpreterRuntime.cpp的InterpreterRuntime::monitorenter函数。

#### 自旋锁以及自适应自旋


在讲这两个锁之前我先讲一个小故事。

小明中午趁着休息时间去银行办事情，但是小明去的时候小明前面有人在办理业务，总不能看到有人办理就回去吧，这样不仅难得跑还浪费了中午的时间，小明决定多等一会儿，看看别人能不能办理完,但是也不能一直等呀，等久了下午上班就会迟到，会被领导批评的。于是小明决定给自己设定一个时间，暂且就为十分钟吧，后面十分钟到了前面还有人在办理，小明决定不等了先回去上班，明天再来。
第二天，小明又来了，但是前面还是有很多人，小明突然有点绝望，小明问了问旁边的哥们等了多久，他说他等了五分钟了，但是今天办理业务的速度比较快，都办了10个人了，刚说着就轮到他了。小明看了下顿时调整了自己的等待时间，觉得自己在多等下肯定能够办理业务了，然后小明就等呀等，想不到前面那个哥们不到一分钟就完事儿了，就轮到小明办理业务了，今天总算没有白跑一趟。

线程阻塞的时候是需要操作系统支持，从用户态切换到核心态的，这个操作给操作系统的并发性能带来了很大的压力(小明跑路很累)。而且很多时候共享数据的锁定状态只会持续很短的一段时间，为了这段时间去挂起和回复线程并不值得，所以我们就让请求锁的那个线程等一下不要放弃处理器的时间，也就是执行一个忙等待(自旋,相当于小明在业务大厅百无聊赖的玩手机,盯着业务办理情况)，但是不能等久了呀，等久了也是很耗费处理器资源的，所以就给自己设定了一个自旋的次数10次(默认)，如果操作了这个次数就挂起了。

后来JDK6中引入了自适应的自旋锁，它会由前一次在同一个锁上的自旋时间以及锁的拥有者状态来决定，如果同一个锁上自旋刚获得，那么就认为这次也有很大几率获取到，就多自旋几次，如果对于某个锁说，自旋很少获取到，就认为没戏，就不自旋了，直接去挂起了。

#### 锁的消除

锁消除是指虚拟机即时编译器在运行时，对一些代码要求同步，但是被检测到不可能存在数据共享竞争的情况的锁进行消除,锁消除的主要判断依据来源于逃逸分析的支持,如果判断在一段代码中，堆上的数据不会逃逸出去从而被其他线程访问到，那么就把他们当成栈数据对待，认为他们是私有的，自然就不用加锁了。比如下面这段代码

```java

public String appendString(String a,String b,String c) {

	StringBuffer sb = new StringBuffer();
	sb.append(a);
	sb.append(b);
	sb.append(c);

	return sb.toString();
}

```

我们都知道StringBuffer是线程安全的,在于它在方法上加了synchronized关键字，此时就持有sb这个对象上的锁了。这段代码就相当于被加了锁，但是我们发现sb作用域被限制在了appendString方法内部，sb的所有引用不会被外部线程访问到，因此这里虽然有锁，但是编译之后就会忽略所有的同步而执行了。

#### 锁的粗化

一般我们使用synchronized，总是推荐将关键字的作用范围限制得越来越小，只有数据竞争的实际范围才进行同步，这样是为了使得需要同步的操作数量尽可能小，其他线程也能尽快拿到锁。但是如果一段代码中反复的对同一个对象加锁，即使没有线程竞争，频繁的进行互斥操作也会导致资源的消耗。

因为如果虚拟机检测到了一段代码频繁的对一个对象加锁，就会将这个锁就行粗化，提升到最外围，比如下面这段代码：

```java
synchronized(stringBuffer) {
	stringBuffer.append(aStr);
	
}

synchronized(stringBuffer) {
	stringBuffer.append(bStr);
}

synchronized(stringBuffer) {
	stringBuffer.append(cStr);
	
}

```

那么此时jvm会将代码优化成下面这样


```java
synchronized(stringBuffer) {
	stringBuffer.append(aStr);
	stringBuffer.append(bStr);
	stringBuffer.append(cStr);
}

```

#### 偏向锁

偏向锁是一种很乐观的情况，针对只有一个线程的情况，比如在一个循环里面，一个线程经常申请锁，释放锁。

在线程进行加锁时，如果该锁对象支持偏向锁，那么 Java 虚拟机会通过 CAS 操作，将当前线程的地址记录在锁对象的标记字段之中,并且将标记字段的最后三位设置为 101(锁标志位还是 01，“是否偏向锁”标志位设置为 1)。

在接下来的运行过程中，每当有线程请求这把锁，Java 虚拟机只需判断锁对象标记字段中：最后三位是否为 101，是否包含当前线程的地址，以及 epoch 值是否和锁对象的类的 epoch 值相同。如果都满足，那么当前线程持有该偏向锁，可以直接返回

一旦出现其它线程竞争锁资源时，偏向锁就会被撤销。偏向锁的撤销需要等待全局安全点，暂停持有该锁的线程，同时检查该线程是否还在执行该方法，如果是，则升级锁，反之则被其它线程抢占。

```c++
void ObjectSynchronizer::fast_enter(Handle obj, BasicLock* lock, bool attempt_rebias, TRAPS) {
 if (UseBiasedLocking) {
 		// 是否到达线程安全点
    if (!SafepointSynchronize::is_at_safepoint()) {
      BiasedLocking::Condition cond = BiasedLocking::revoke_and_rebias(obj, attempt_rebias, THREAD);
      if (cond == BiasedLocking::BIAS_REVOKED_AND_REBIASED) {
        return;
      }
    } else {
      assert(!attempt_rebias, "can not rebias toward VM thread");

      // 撤销偏向锁
      BiasedLocking::revoke_at_safepoint(obj);
    }
    assert(!obj->mark()->has_bias_pattern(), "biases should be revoked by now");
 }

 slow_enter (obj, lock, THREAD) ;
}

```

由于biasedLocking.cpp中方法太长了，我就不引入了。


MarkOop中还有一个epoch值，这个值表示第几代锁，这是什么意思呢？

当从撤销偏向锁说起。当请求加锁的线程和锁对象标记字段保持的线程地址不匹配时(同时也需要判断epoch是否相等，不等，那么当前该线程可以将锁重新偏向自己)，Java 虚拟机需要撤销该偏向锁。这个撤销过程非常麻烦，它要求持有偏向锁的线程到达安全点，再将偏向锁替换成轻量级锁。

具体的做法便是在每个类中维护一个 epoch 值，你可以理解为第几代偏向锁。

当设置偏向锁时，Java 虚拟机需要将该 epoch 值复制到锁对象的标记字段中。在宣布某个类的偏向锁失效时，Java 虚拟机实则将该类的 epoch 值加 1，表示之前那一代的偏向锁已经失效。

而新设置的偏向锁则需要复制新的 epoch 值。为了保证当前持有偏向锁并且已加锁的线程不至于因此丢锁，Java 虚拟机需要遍历所有线程的 Java 栈，找出该类已加锁的实例，并且将它们标记字段中的 epoch 值加 1。该操作需要所有线程处于安全点状态。如果总撤销数超过另一个阈值（对应 Java 虚拟机参数 -XX:BiasedLockingBulkRevokeThreshold，默认值为 40），那么 Java 虚拟机会认为这个类已经不再适合偏向锁。此时，Java虚拟机会撤销该类实例的偏向锁，并且在之后的加锁过程中直接为该类实例设置轻量级锁。

> 如果你理解不了第几代的意思，你可以把它当成一个版本号或者时间戳都是可以的
> 由于偏向锁是默认打开的，如果存在大量线程竞争资源的情况，可以关闭偏向锁: -XX:-UseBiasedLocking


#### 轻量级锁

如果此时又有另外一个线程B需要进入临界区执行代码，但是此时锁对象的Mark Word中是偏向线程A的，这个时候线程B是不能进入临界区。
此时会将锁对象修改为无锁状态，然后在各自的栈分配一个空间，叫做Lock Record(锁记录),把锁对象的Mark Word在A,B两个线程各自复制一份，叫做Displaced Mard Word.
然后将Lock Record的地址使用CAS放到了Mark Word中，并且将锁标志位修改为00，哪个线程修改成功，就代表获取到了锁，可以进入临界区。没有获得锁的线程自旋几次等待其他线程释放锁。

```c++
void ObjectSynchronizer::slow_enter(Handle obj, BasicLock* lock, TRAPS) {
  markOop mark = obj->mark();
  assert(!mark->has_bias_pattern(), "should not see bias pattern here");

	// 判断是否是无锁状态
  if (mark->is_neutral()) {
    lock->set_displaced_header(mark);
    if (mark == (markOop) Atomic::cmpxchg_ptr(lock, obj()->mark_addr(), mark)) {
      TEVENT (slow_enter: release stacklock) ;
      return ;
    }
    
  } else
  if (mark->has_locker() && THREAD->is_lock_owned((address)mark->locker())) {
    assert(lock != mark->locker(), "must not re-lock the same lock");
    assert(lock != (BasicLock*)obj->mark(), "don't relock with same BasicLock");
    lock->set_displaced_header(NULL);
    return;
  }

  lock->set_displaced_header(markOopDesc::unused_mark());
  ObjectSynchronizer::inflate(THREAD, obj())->enter(THREAD);
}

```



#### 重量级锁
A,B线程执行，相安无事，即便哪一个获取到了锁也很快就会让出来，另外一个不会等待太久，但是有一天，某个线程自旋了很多次还是没有获取到锁，这个时候就需要操作系统的帮助，这个将轻量级锁升级为重量级锁，同时将标志位修改为10，阻塞其他未获取到锁的线程。

在这个状态下，未抢到锁的线程都会进入 Monitor，之后会被阻塞在 _WaitSet 队列中。

### 总结

java对于锁的优化是自动的，不需要我们做什么，编译器在编译的时候会分析代码对代码进行优化。

JVM中关于锁的优化大抵如是了，其实synchronized并不比lock慢多少，甚至在某些方面可能还比lock快，而且JDK8中ConcurrentHashMap也将之前的分段锁改为了Synchronized实现。




