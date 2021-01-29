---
title: 创建线程的几种主流方式
date: 2021-01-21 19:19:36
tags: [多线程,java]
---


### 继承Thread类

继承Thread类，并重写它的run方法，就可以创建一个线程了，当然线程是如何真正被启动，可以参考我之前的 [为什么start方法才能启动线程,而run不行？](https://juejin.cn/post/6858058928467968008)

```java
class ThinkThread extends Thread {
  @Override
  public void run() {
      System.out.println("think123");
  }
}

new ThinkThread().start();

```

<!--more-->

### 实现Runnable接口

```java
 new Thread(() -> 
  System.out.println("实现了Runnable接口")
).start();
```

构造Thread时传入Runnable类型的参数，也可以创建一个线程

### 通过线程池创建线程

```java
// 提交 Runnable 任务
Future<?> submit(Runnable task);

// 提交 Callable 任务
<T> Future<T> submit(Callable<T> task);

// 提交 Runnable 任务及结果引用。 future.get()==result
<T> Future<T> submit(Runnable task, T result);
```

Runnable类型的参数和Callable类型的参数不同之处在于Runnable 接口的 run() 方法是没有返回值的，所以 submit(Runnable task)这个方法返回的 Future 仅可以用来断言任务已经结束了，类似于 Thread.join()。 而Callable是一个接口，它有一个call()方法，这个方法是有返回值的，这个可以通过 future.get() l获取任务执行结果

```java
// 建议手动创建线程池，这里只是为了举例
ExecutorService service = Executors.newFixedThreadPool(1);

Future future = service.submit(() -> "think123");

System.out.println(future.get());

```

### 通过FutureTask创建线程

FutureTask继承了Runnable和Future接口,所以我们可以将FutureTask对象作为任务提交到线程池执行，也可以直接被Thread执行，而且还可以获取到任务执行结果。

```java
FutureTask task = new FutureTask(() -> "666");

Thread t1 = new Thread(task);
t1.start();;

// 阻塞main线程，直到t1执行完成
System.out.println(task.get());
```

FutureTask的源码很简单，当执行run方法时，会将执行的结果保存在内部变量 outcome 中，即便是抛出了异常，此时也会将异常记录到outcome中。

当调用 get 方法时，如果还未执行完成，则会阻塞调用方。执行完成后会将正常的结果返回，如果call方法中抛出了异常，则将其封装成 ExecutionException 抛出。


### 总结

上面介绍了Java中常用的创建线程执行的方式，可以发现，实际上都是通过 创建Thread来执行的，实际上也是可以算作一种，如果面试官问题，你可以先装一下说之后一种，然后峰回路转，给他说说为什么之后一种。

> 当然还有其他方式，比如TimerTask, Quartz Job但是实际上都是类似的。