---
title: 分析Java Object中的wait/notify
date: 2019-10-10 11:11:59
tags: Java
---

在Java的Object类中有2个我们不怎么常用(框架中用的更多)的方法：wait()与notify()或notfiyAll()，这两个方法主要用于多线程间的协同处理，即控制线程之间的等待、通知、切换及唤醒。

首先了解下线程有哪几种状态,Java的[Thread.State](https://docs.oracle.com/javase/7/docs/api/java/lang/Thread.State.html)中定义了线程的6种状态,分别如下：

1. NEW    未启动的,不会出现在dump文件中(可以通过jstack命令查看线程堆栈)
2. RUNNABLE    正在JVM中执行的
3. BLOCKED    被阻塞,等待获取监视器锁进入synchronized代码块或者在调用Object.wait之后重新进入synchronized代码块
4. WAITING    无限期等待另一个线程执行特定动作后唤醒它,也就是调用Object.wait后会等待拥有同一个监视器锁的线程调用notify/notifyAll来进行唤醒
5. TIMED_WAITING    有时限的等待另一个线程执行特定动作
6. TERMINATED    已经完成了执行

<!--more-->

>从操作系统层面上来讲，一个进程从创建到消亡期间，最常见的进程状态有以下几种
> 新建态 ： 从程序映像到进程映像的转变，还没有加入到就绪队列中
> 就绪态 : 进程运行已万事俱备，正等待调度执行
> 运行态 : 进程指令正在被执行
> 阻塞态 : 进程正在等待一个时间操作完成，例如I/O操作
> 完成态 : 进程运行结束，它的资源已经被释放，供其他活动进程使用

接下来我们分析下面的代码

```java
public class WaitNotify {

  public static void main(String[] args) {

    Object lock = new Object();
    
    // thread1
    new Thread(() -> {

        System.out.println("thread1 is ready");

        try {
            Thread.sleep(2000);
        } catch (InterruptedException e) {

        }
        synchronized (lock) {
            lock.notify();
            System.out.println("thread1 is notify,but not exit synchronized");
            System.out.println("thread1 is exit synchronized");
        }


    }).start();

    // thread2
    new Thread(() -> {
        System.out.println("thread2 is ready");

        try {
            Thread.sleep(1000);
        } catch (InterruptedException e) {

        }
        synchronized (lock) {
            try {
                System.out.println("thread2 is waiting");
                lock.wait();
                System.out.println("thread2 is awake");
            } catch (InterruptedException e) {
            }
        }
    }).start();
  }
}
```

上面的代码运行结果如下

```
thread1 is ready
thread2 is ready
thread2 is waiting
thread1 is notify,but not exit synchronized
thread 1 is exit synchronized
thread2 is awake
```

看到这里你会发现平平无奇，好像也没什么特殊的事情,但是如果深入分析下，就会发现以下的问题

1. 为何调用wait或者notify一定要加synchronized，不加行不行？
2. thread2中调用了wait后，当前线程还未退出同步代码块，其他线程(thread1)能进入同步块吗？
3. 为何调用wait()有可能抛出InterruptedException异常
4. 调用notify/notifyAll后等待中的线程会立刻唤醒吗？
5. 调用notify/notifyAll是随机从等待线程队列中取一个或者按某种规律取一个来执行？
6. wait的线程是否会影响系统性能？

针对上面的问题，我们逐个分析下

#### 为何调用wait或者notify一定要加synchronized，不加行不行？

如果你不加，你会得到下面的异常

```java
Exception in thread "main" java.lang.IllegalMonitorStateException
```
>在[JVM源代码中](https://github.com/unofficial-openjdk/openjdk/blob/jdk8u%2Fjdk8u/hotspot/src/share/vm/runtime/objectMonitor.cpp#L1425)首先会检查当前线程是否持有锁,如果没有持有则抛出异常

其次为什么要加,也有比较广泛的讨论，首先wait/notify是为了线程间通信的，为了这个通信过程不被打断，需要保证wait/notify这个整体代码块的原子性，所以需要通过synchronized来加锁。


#### thread2中调用了wait后，当前线程还未退出同步代码块，其他线程(thread1)能进入同步块吗？

wait在处理过程中会临时释放同步锁(如果不释放其他线程没有机会抢)，不过需要注意的是当其他线程调用notify唤起这个线程的时候，在wait方法退出之前会重新获取这把锁，只有获取了这把锁才会继续执行,这也和我们的结果相符合,输出了`thread2 is awake`,
其实想想也容易理解,synchronized的代码实际上是被被monitorenter和monitorexit包围起来的。当我们调用wait的时候，会释放锁,调用monitorexit的时候也会释放锁,那么当thread2被唤醒的时候必然重新获取到了锁(objectMonitor::enter)。

其实从jdk源代码的[ObjectMonitor::wait方法](https://github.com/unofficial-openjdk/openjdk/blob/jdk8u%2Fjdk8u/hotspot/src/share/vm/runtime/objectMonitor.cpp#L1463)可以一窥究竟，首先会放弃已经抢到的锁(exit(self)),而放弃锁的前提是获取到锁

而在notify方法中会选取一个线程获得cpu执行权，在去竞争锁，如果没有竞争到则会进入休眠。

如果调用的wait(200)这种代码,那么会在200ms后将线程从waiting set中移除并允许其重新竞争锁，需要注意的是notify方法并不会释放所持有的monitor


#### 为何调用wait()有可能抛出InterruptedException异常

当我们调用了某个线程的interrupt方法，对应的线程会抛出这个异常，wait方法也不希望去破坏这种规则，因此就算当前线程因为wait一直在阻塞。当某个线程希望它起来继续执行的时候，它还是得从阻塞态恢复过来，因此wait方法被唤醒起来的时候会去检测这个状态，当有线程interrupt了它的时候，它就会抛出这个异常从阻塞状态恢复过来。

这里有两点要注意：
1. 如果被interrupt的线程只是创建了，并没有start，那等他start之后进入wait态之后也是不能会恢复的
2. 如果被interrupt的线程已经start了，在进入wait之前，如果有线程调用了其interrupt方法，那这个wait等于什么都没做，会直接跳出来，不会阻塞,下面的代码演示了这种情况

```java
Thread thread2 = new Thread(() -> {
    System.out.println("thread2 is ready");

    try {
        Thread.sleep(1000);
    } catch (InterruptedException e) {

    }
    synchronized (lock) {
        try {
            System.out.println("thread2 is waiting");
            lock.wait();
            System.out.println("thread2 is awake");
        } catch (InterruptedException e) {
            System.out.println(e);
        }
    }
});

  // main
  thread2.start();

  Thread.sleep(1000);
  thread2.interrupt();

```

上面的代码输出结果如下

```
thread2 is ready
thread2 is waiting
java.lang.InterruptedException
```

#### 调用notify/notifyAll后等待中的线程会立刻唤醒吗？

hotspot真正的实现是退出同步代码块的时候才会去真正唤醒对应的线程，不过这个也是个默认策略，也可以改的，在notify之后立马唤醒相关线程。
这个也可从jdk源代码的objectMonitor类objectMonitor::notify方法中看到.在调用notify时的默认策略是Policy == 2（这个值是源码中的初值，可以通过-XX:SyncKnobs来设置）

其实对于Policy(1、2、3、4)都是将objectMonitor的ObjectWaiter集合中取出一个等待线程，放入到_EntryList（blocked线程集合，可以参与下次抢锁），只是放入_EntryList的策略不一样，体现为唤醒wait线程的规则不一样。

对于默认策略notify在将一个等待线程放入阻塞线程集合之后就退出，因为同步块还没有执行完monitorexit，锁其实还未释放，所以在打印出“thread1 is exit synchronized!”的时候，thread2线程还是blocked状态（因为thread1还没有退出同步块）。

这里可以发现，对于不在Policy中的情况，会直接将一个ObjectWaiter进行unpark唤醒操作，但是被唤醒的线程是否立即获取到了锁呢？答案是否定的。



#### 调用notify/notifyAll是随机从等待线程队列中取一个或者按某种规律取一个来执行？

我们自己实现可能一个for循环就搞定了，不过在jvm里实现没这么简单，而是借助了monitor_exit，上面我提到了当某个线程从wait状态恢复出来的时候，要先获取锁，然后再退出同步块，所以notifyAll的实现是调用notify的线程在退出其同步块的时候唤醒起最后一个进入wait状态的线程，然后这个线程退出同步块的时候继续唤醒其倒数第二个进入wait状态的线程，依次类推，同样这是一个策略的问题，jvm里提供了挨个直接唤醒线程的参数,这里要分情况：

1. 如果是通过notify来唤起的线程，那先进入wait的线程会先被唤起来
2. 如果是通过nootifyAll唤起的线程，默认情况是最后进入的会先被唤起来，即LIFO的策略

#### wait的线程是否会影响系统性能？

这个或许是大家比较关心的话题，因为关乎系统性能问题，wait/nofity是通过jvm里的park/unpark机制来实现的，在linux下这种机制又是通过pthread_cond_wait/pthread_cond_signal来玩的，因此当线程进入到wait状态的时候其实是会放弃cpu的，也就是说这类线程是不会占用cpu资源，也不会影响系统加载。


#### 什么是监视器(monitor)

Java中每一个对象都可以成为一个监视器（Monitor）, 该Monitor由一个锁（lock）, 一个等待队列（waiting queue ）, 一个入口队列( entry queue)组成.
对于一个对象的方法， 如果没有synchonized关键字， 该方法可以被任意数量的线程，在任意时刻调用。
对于添加了synchronized关键字的方法，任意时刻只能被唯一的一个获得了对象实例锁的线程调用。

![java监视器](/images/java_monitor.bmp)


**进入区(Entry Set):** 表示线程通过 synchronized要求获得对象锁，如果获取到了，则成为拥有者，如果没有获取到在在进入区等待，直到其他线程释放锁之后再去竞争(谁获取到则根据)

**拥有者(Owner):** 表示线程获取到了对象锁，可以执行synchronized包围的代码了

**等待区(Wait Set):** 表示线程调用了wait方法,此时释放了持有的对象锁，等待被唤醒(谁被唤醒取得监视器锁由jvm决定)

#### 关于sleep

它是一个静态方法，一般的调用方式是Thread.sleep(2000),表示让当前线程休眠2000ms,并不会让出监视器，这一点需要注意。