title: 关于wait,notify,join,sleep的用法说明
date: 2018-07-16 20:21:05
tags: [多线程]
categories: [多线程]
description: 一说到多线程，那么必然离不开Object中的wait,notify,notifyAll,join以及Thread中的sleep等方法,虽然这些是耳熟能详，但是你真的会用吗？你知道如何使用吗？你知道这里面涉及到的知识点吗？
keywords: wait,notify,notifyAll,join,sleep,synchronized
---
提到多线程总是绕不开wait,notify,notifyAll这些操作,在往下看之前,问问自己真的知道它们怎么使用吗?

### 什么是线程
进程（process）和线程（thread）是操作系统的基本概念，但是它们比较抽象，我举个例子来帮助理解。
唐僧对悟空说:"悟空,你要好好保护我，取经路上艰辛呀,但是为师现在有点饿了，你去给我化点斋饭吧?",悟空心想，我又要保护你，又要给你化缘，我很可咋办呀？(同一时间一个CPU只能处理一个任务，这里的悟空就是一个进程),突然悟空灵光一闪，我用我的猴毛变一个我，然后去给师傅老人家化缘不就行了，这样我也可以保护师傅(分出去的猴毛就是一个线程)。
所以，线程（英语：thread）是操作系统能够进行运算调度的最小单位。它被包含在进程之中，是进程中的实际运作单位。一条线程指的是进程中一个单一顺序的控制流，一个进程中可以并发多个线程，每条线程并行执行不同的任务。

### 线程的状态

Java的[Thread.State](https://docs.oracle.com/javase/7/docs/api/java/lang/Thread.State.html)中定义了线程的6种状态,分别如下：

1. NEW    未启动的,不会出现在dump文件中(可以通过jstack命令查看线程堆栈)
2. RUNNABLE    正在JVM中执行的
3. BLOCKED    被阻塞,等待获取监视器锁进入synchronized代码块或者在调用Object.wait之后重新进入synchronized代码块
4. WAITING    无限期等待另一个线程执行特定动作后唤醒它,也就是调用Object.wait后会等待拥有同一个监视器锁的线程调用notify/notifyAll来进行唤醒
5. TIMED_WAITING    有时限的等待另一个线程执行特定动作
6. TERMINATED    已经完成了执行

### 和多线程相关的方法以及关键字

wait,notify,notifyAll,sleep,join以及synchronized

### 错误的使用wait


```java
public class ThreadTest {

public static void main(String[] args) {
	ThreadTest tt = new ThreadTest();
		try {
			tt.wait();
		} catch (InterruptedException e) {
			e.printStackTrace();
		}
	}
}
```

当我们运行的时候会看到如下的错误：

```java
Exception in thread "main" java.lang.IllegalMonitorStateException
```

这是为什么呢？官方API说的很清楚了，在调用这个方法的时候当前线程必须拥有这个对象的 **监视器**。调用了wait方法之后这个线程会释放监视器锁直到调用notify/notifyAll方法。

### 如何获取监视器？

通常我们使用synchronized关键字来获取监视器，我们可能会经常看到这样的代码

```java

public static synchronized void execute1() {
    System.out.println("synchronized method test");
}

public static Boolean execute2() {
    synchronized (ThreadTest.class) {
        System.out.println("synchronized code block test");
    }
       return true;
    }

    public  static  void  main(String[] args) {
       execute1();
       System.out.println(execute2());
    }

```

这就是synchronized的用法,一般使用它用来修饰语句或者修饰方法,对于synchronized语句当Java源代码被javac编译成bytecode的时候，会在同步块的入口位置和退出位置分别插入monitorenter和monitorexit字节码指令.而synchronized方法则会被翻译成普通的方法调用和返回指令如:invokevirtual、return指令，在VM字节码层面并没有任何特别的指令来实现被synchronized修饰的方法，而是在Class文件的方法表中将该方法的access_flags字段中的synchronized标志位置1，表示该方法是同步方法并使用调用该方法的对象或该方法所属的Class在JVM的内部对象表示Klass做为锁对象这里就不对synchronized原理做深入说明了。

![反编译代码](/images/synchro_compile_code.png)

### 为什么使用synchronized
synchonized用于实现多线程的同步操作，一般多线程协作过程中,都会操作共享资源,可能是某个类,比如HashMap也可能是硬盘文件,数据库资源等等，不正当的操作方式会导致结果难以预料。线程安全需要保证几个基本特征：

1. 原子性，简单说就是相关操作不会中途被其他线程干扰，一般通过同步机制实现

2. 可见性，是一个线程修改了某个共享变量，其状态能够立即被其他线程知晓，通常被解释为将线程本地状态同步到主内存上，volatile就可以保证可见性

3. 有序性，是保证线程内串行语义，避免[指令重排](https://www.cs.umd.edu/~pugh/java/memoryModel/jsr-133-faq.html)

而synchronized能保证这些特性，所以它也是保证线程安全的一种手段，但是不要滥用，容易导致死锁，关于线程安全你可以看看我的这篇[文章](http://generalthink.github.io/2016/09/21/talk-thread-safe/),当然今天的重点不是这个，毕竟我们聊的是wait。

### 什么是监视器(monitor)

Java中每一个对象都可以成为一个监视器（Monitor）, 该Monitor由一个锁（lock）, 一个等待队列（waiting queue ）, 一个入口队列( entry queue).
对于一个对象的方法， 如果没有synchonized关键字， 该方法可以被任意数量的线程，在任意时刻调用。
对于添加了synchronized关键字的方法，任意时刻只能被唯一的一个获得了对象实例锁的线程调用。

![java监视器](/images/java_monitor.bmp)


**进入区(Entry Set):** 表示线程通过 synchronized要求获得对象锁，如果获取到了，则成为拥有者，如果没有获取到在在进入区等待，直到其他线程释放锁之后再去竞争(谁获取到则根据)

**拥有者(Owner):** 表示线程获取到了对象锁，可以执行synchronized包围的代码了

**等待区(Wait Set):** 表示线程调用了wait方法,此时释放了持有的对象锁，等待被唤醒(谁被唤醒取得监视器锁由jvm决定)

### 正确的使用wait/notify

之前也说了wait/notify是为了处理线程之间协作的问题的，那么既然是协作必然涉及到了对公共资源的操作。所以我们举一个生产者消费者的例子。

```java
import java.util.LinkedList;
import java.util.Queue;

public class ThreadTest {

public static void main(String[] args) {
	Queue<Integer> queue = new LinkedList<>();
	Thread producer = new Thread(new Producer(queue,20));
	Thread consumer = new Thread(new Consumer(queue,20));

	producer.start();
	consumer.start();

	}
}

class Producer implements Runnable {

    Queue<Integer> queue;
    Integer maxSize = 10;
    Integer maxTime;

    public Producer(Queue<Integer> queue, Integer maxTime) {
        this.queue = queue;
        this.maxTime = maxTime;
    }

    @Override
    public void run() {
        int i = 0;
        while (i <= maxTime) {
            synchronized (queue) {
                while (queue.size() >= maxSize) {
                    try {
                        System.out.println("waiting producer");
                        queue.wait();
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                }
                // 产生一个数据
                int d = (int) (Math.random() * 100);
                queue.add(d);
                System.out.println("producer queue size is " + queue.size() + ",produce data " + d);

                queue.notify();

                i++;
            }
        }

    }
}

class Consumer implements Runnable {

	Queue<Integer> queue;
	Integer maxTime;

	public Consumer(Queue queue,Integer maxTime) {
		this.queue = queue;
		this.maxTime = maxTime;
	}

	@Override
	public void run() {
		int i = 0;
		while(i <= maxTime) {
			synchronized (queue) {
				while(queue.isEmpty()) {
					try {
						queue.wait();
					} catch (InterruptedException e) {
						e.printStackTrace();
					}
				}

				System.out.println("consume data " + queue.poll());
				//去唤醒生产者
				queue.notify();
				i++;
			}
		}
	}
}

```

输出结果如下：

```
producer queue size is 1,produce data  99
producer queue size is 2,produce data  55
producer queue size is 3,produce data  59
producer queue size is 4,produce data  78
producer queue size is 5,produce data  99
producer queue size is 6,produce data  30
producer queue size is 7,produce data  40
producer queue size is 8,produce data  28
producer queue size is 9,produce data  52
producer queue size is 10,produce data  57
queue is full,waiting produce
consume data 99
consume data 55
consume data 59
consume data 78
consume data 99
consume data 30
consume data 40
consume data 28
consume data 52
consume data 57
producer queue size is 1,produce data  29
producer queue size is 2,produce data  72
producer queue size is 3,produce data  8
producer queue size is 4,produce data  66
producer queue size is 5,produce data  27
producer queue size is 6,produce data  37
producer queue size is 7,produce data  27
producer queue size is 8,produce data  27
producer queue size is 9,produce data  4
producer queue size is 10,produce data  87
queue is full,waiting produce
consume data 29
consume data 72
consume data 8
consume data 66
consume data 27
consume data 37
consume data 27
consume data 27
consume data 4
consume data 87
producer queue size is 1,produce data  79
consume data 79
```

上面的wait的用法基本可以简化成为这样

```java
synchronized(queue) {//使当前线程获取到queue对象的监视器

	//让拥有queue对象监视器的线程进行等待(让出monitor使用权)直到被唤醒(notify/notifyAll)
    queue.wait();

    doSomething();
}
```

所以，wait和notify/notifyAll基本上是成对出现的,
wait(0)=wait()后的线程必须要由notify/notifyAll唤醒才能继续去竞争锁，从而运行wait之后的代码.如果调用的wait(200)这种代码,那么会在200ms后将线程从waiting set中移除并允许其重新竞争锁，需要注意的是notify方法并不会释放所持有的monitor.

### join的使用

先来看看join的源码，咋在接着分析

```java
public final synchronized void join(long millis)
	throws InterruptedException {
	long base = System.currentTimeMillis();
	long now = 0;

	if (millis < 0) {
		throw new IllegalArgumentException("timeout value is negative");
	}

	if (millis == 0) {
		while (isAlive()) {
			wait(0);
		}
	} else {
		while (isAlive()) {
			long delay = millis - now;
			if (delay <= 0) {
				break;
			}
			wait(delay);
			now = System.currentTimeMillis() - base;
		}
	}
}
```

如果我们写了这样的一段代码

```java
public class ThreadTest implements Runnable{

	public static void main(String[] args) {
		Thread th = new Thread(new ThreadTest());
		th.start();
		try {
			th.join();
		} catch (InterruptedException e) {
			e.printStackTrace();
		}
		System.out.println("main thread execute over");
	}

	@Override
	public void run() {
		try {
			System.out.println("sleep 3s");
			Thread.sleep(3000);
			System.out.println("wake up");
		} catch (Exception e) {
			e.printStackTrace();
		}
	}
}
```

那么输出结果是：

```java
sleep 3s
wake up
main thread execute over
```

如果将th.join修改为th.join(2000)那么main thread会在sleep 3s之后打印。
上面我们贴出来了join的源码，其实我觉得th.join()可以修改成为这样

```java
synchronized(th) {
while(th.isAlive()) {
    th.wait();
    break;
}
}

```

synchronized关键字是加在方法上面的，这个方法并不是一个static方法，所以此时获取到的是线程实例的监视器，在上面这段代码中获取到的就是th这实例的监视器。由于是main线程运行的th.join()这段代码所以会导致main线程进行等待(等待时间可以通过添加调用不同的方法来决定，比如wait(2000),join(2000)),上面我们发现join方法其实内部也是使用的wait，但是是哪里notify的呢？官方api说明，在线程结束后会隐含的调用notifyAll方法，所以才会有输出，如果把锁加在其他对象(比如new Object)上，会发现一直等待，应用程序无法退出。

### 关于sleep

它是一个静态方法，一般的调用方式是Thread.sleep(2000),表示让当前线程休眠2000ms,并不会让出监视器，这一点需要注意。

### 总结

1. wait/notify基本上是成对出现，有时候在代码中使用不当会导致程序出现异常效果
2. 关于monitor的概念需要了解清楚，wait/join方法会让出monitor,notify不会


