title: 关于锁使用的补充说明
date: 2018-07-24 21:26:25
tags: [jvm,多线程]
categories: 多线程
description: 在涉及到临界区的时候，就不得到谈到synchronized,又不得不谈到notify,这个也是对锁的优化那一章的一些补充说明。
keywords: synchronized,jvm,多线程,临界区
---

上一篇文章我讲了下synchronized关于锁优化的问题，但是还是有人不太清楚临界区到底是什么，也对notify的机制有点疑惑，这篇文章就来分析一下。

### 什么是临界区

我们都知道Synchroinzed可以修饰代码块，实例方法,静态方法，区别只是持有的对象锁不一样罢了，持有了这把锁你就可以为所欲为了，就好比很多人去抢厕所，如果大家都在一个厕所里面上，那么很明显会发生很多不可描述的事儿,所以就需要去抢厕所的使用，你抢到了然后你进去厕所把门反锁了，这个时候别人进不来，厕所里面的公共资源你想怎样用就怎样用，只有等你出来之后这个时候其他人才能去抢厕所。如果在你准备上厕所的时候你发现没有厕纸了，这个时候你得要先去拿纸，让别人先用厕所了。

上面的例子也很好的说明了临界区(上面故事的厕所，也就是公共资源),当多个线程去竞争的时候获取锁的这个线程就可以安全的执行，可以保证原子性，一致性，顺序性(JMM)。获取锁这个操作是通过monitorenter,退出临界区的时候是执行monitorexit指令,此时会释放持有的锁。这个时候线程可以继续去竞争,至于是谁竞争上就取决于操作系统了,线程的优先级并不一定能起作用，因为调度是取决于操作系统，不同的操作系统的优先级不一样，可能操作系统总共只有三个级别，但是java中是从1到10的，那么必然会导致有重复，所以调度不要完全依赖优先级。

当遇到没有带厕纸的时候，其实这也对应了一个问题，就是在执行某些操作的时候可能需要等待其他条件先满足在执行，这个时候你就会让出这把锁，让其他线程去执行，等到条件满足了之后你在去运行。

需要注意的是当操作系统分配给我们的时间片用完了的时候这个时候并不会让出锁(Thread.sleep)，而是会等待操作系统的调度，等到下一次执行。


### 调用了notify(All)后谁会被唤醒？

这里存在**两种情况**

1. 如果是通过notify来唤起的线程，那先进入wait的线程会先被唤起来
2. 如果是通过nootifyAll唤起的线程，默认情况是最后进入的会先被唤起来，即LIFO的策略

验证这个问题很简单，看看下面这段代码

```java
public class NotifyTest {

    public static void main(String[] args) {
        Object lock  = new Object();
        Thread p = new Thread(new Producer(lock));
        Thread con = new Thread(new Consumer(lock));

        p.start();
        con.start();


    }

}

class Producer implements  Runnable {

    Object lock;

    public Producer(Object obj) {
        this.lock = obj;
    }

    @Override
    public void run() {

        synchronized (lock) {
            try {
                System.out.println("producer will wait");
                lock.wait();
                System.out.println("producer wait over");
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }
}



class Consumer implements  Runnable {

    Object lock;

    public Consumer(Object obj) {
        this.lock = obj;
    }
    @Override
    public void run() {

        try {
            //模拟producer先执行
            Thread.sleep(2000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        synchronized (lock) {
            System.out.println("consumer will notify");
            lock.notify();
            System.out.println("producer notify over");
        }
    }
}

```

输出结果如下:

```
producer will wait
consumer will notify
producer notify over
producer wait over

```

这个结果证明了我刚才说的，只有在退出同步代码块之后才会释放锁，而不是在调用了notify(All)方法就马上去唤醒，但是这只是一个默认策略，也是可以修改的。

### wait的线程是否会占用cpu?

wait/nofity是通过jvm里的park/unpark机制[wait/notify底层原理](https://github.com/openjdk-mirror/jdk7u-hotspot/blob/50bdefc3afe944ca74c3093e7448d6b889cd20d1/src/share/vm/runtime/objectMonitor.cpp#L1507)
来实现的，在linux下这种机制又是通过pthread_cond_wait/pthread_cond_signal来完成的，pthread_cond_wait其思想就是先释放锁，让其他线程可以执行，然后进入睡眠，等待被唤醒，被唤醒之后重新加锁。
因此当线程进入到wait状态的时候其实是会放弃cpu的，也就是说这类线程是不会占用cpu资源。
