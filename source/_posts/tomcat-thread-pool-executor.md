---
title: 我丢,线程池原来还可以优先扩容核心线程数？
date: 2020-08-12 15:21:15
tags: java
---

在之前的《为什么阿里建议你不要使用Executors来创建线程池？》文章中深入分析了一下线程池,这里重温下它的主要流程:

1. 提交一个任务，如果线程池中的工作线程数小于corePoolSize,则新建一个工作线程执行任务
2. 如果线程池当前的工作线程已经等于了corePoolSize,则将新的任务放入到工作队列中正在执行
3. 如果工作队列已经满了,并且工作线程数小于maximumPoolSize,则新建一个工作线程来执行任务
4. 如果当前线程池中工作线程数已经达到了maximumPoolSize,而新的任务无法放入到任务队列中,则采用对应的策略进行相应的处理(默认是拒绝策略)
5. 当线程数大于核心线程数时，线程等待 keepAliveTime 后还是没有任务需要处理的话，收缩线程到核心线程数。(传入 true 给 allowCoreThreadTimeOut 方法,来让线程池在空闲的时候同样回收核心线程。)


在JDK自带的策略中有一个CallerRunsPolicy策略，很容易被忽视,当任务队列已经满了的时候这个任务会被调用线程池的线程执行。
比如我们在tomcat线程中调用了线程池，当线程池的队列满了之后,会将这个任务交给tomcat线程执行。这个时候就会影响其他同步执行的线程，甚至可能把线程池搞崩溃。



我们还可以通过一些手段来修改线程池的默认行为，比如回收核心线程。或者声明线程池后立即调用 prestartAllCoreThreads 方法，来启动所有核心线程。


我们发现核心线程数只有在工作队列满了之后才会扩容,那么能不能先扩容核心线程,等到达到最大线程数之后再加入工作队列呢？让线程池更弹性,优先开启更多线程呢？

当然可以，tomcat中的ThreadPoolExecutor就是就做了这样的优化。

tomcat的线程池在创建的时候会先启动所有的核心线程(prestartAllCoreThreads),并且会优先扩容线程数。


tomcat中的`ThreadPoolExecutor`继承自`java.util.concurrent.ThreadPoolExecutor`,并且任务队列使用的是`TaskQueue`(继承自`LinkedBlockingQueue<Runnable>`)


```java

public class ThreadPoolExecutor extends java.util.concurrent.ThreadPoolExecutor {

public ThreadPoolExecutor(int corePoolSize, int maximumPoolSize, long keepAliveTime, 
  TimeUnit unit, BlockingQueue<Runnable> workQueue, RejectedExecutionHandler handler) {
      super(corePoolSize, maximumPoolSize, keepAliveTime, unit, workQueue, handler);
      // 调用父类的方法,先启动所有的核心线程
      prestartAllCoreThreads();
    }
  public void execute(Runnable command) {
    if (command == null)
      throw new NullPointerException();

    int c = ctl.get();
    // 1. 如果工作线程数小于核心线程数(corePoolSize),则创建一个工作线程执行任务
    if (workerCountOf(c) < corePoolSize) {
      if (addWorker(command, true))
          return;
      c = ctl.get();
    }

    // 2. 如果当前是running状态，并且任务队列能够添加任务
    if (isRunning(c) && workQueue.offer(command)) {
      int recheck = ctl.get();

      // 2.1 如果不处于running状态了(使用者可能调用了shutdown方法),
      // 则将刚才添加到任务队列的任务移除
      if (! isRunning(recheck) && remove(command))
          reject(command);
       // 2.2 如果当前没有工作线程,
       // 则新建一个工作线程来执行任务(任务已经被添加到了任务队列)
      else if (workerCountOf(recheck) == 0)
          addWorker(null, false);
    }

    // 3. 队列已经满了的情况下，则新启动一个工作线程来执行任务
    else if (!addWorker(command, false))
      reject(command);
  }

}


public class TaskQueue extends LinkedBlockingQueue<Runnable> {

  // 通过setParent方法设置线程池
  private volatile ThreadPoolExecutor parent = null;

  @Override
  public boolean offer(Runnable o) {
    
    if (parent==null) return super.offer(o);

    // 1. 线程数已经扩容到了最大线程数,此时正常加入队列
    if (parent.getPoolSize() == parent.getMaximumPoolSize()) return super.offer(o);

    // 2. 存在空闲线程将其加入到队列中
    if (parent.getSubmittedCount()<=(parent.getPoolSize())) return super.offer(o);

    // 3. 核心线程数少于最大线程数,不加入队列,而是会创建一个新的工作线程
    if (parent.getPoolSize()<parent.getMaximumPoolSize()) return false;
    
    // 加入队列
    return super.offer(o);
  }
}
```

比如核心线程数设置为1,最大线程数设置为2,每个任务执行时间是5s,当第一个任务提交之后,submittedCount=1,会创建一个工作线程执行任务，poolSize变成1(查看execute方法的第一步)。

此时第二个任务提交了，submittedCount的值为2。不符合offer方法中第一和第二个判断,但是符合第三个判断,返回false,表示加入队列失败(表示队列已满)

此时在回到execute的第三个条件判断,直接启动一个新的工作线程来执行任务。

这样就做到了优先扩容到最大线程数。来不及处理的多余任务才会放入到队列中。
