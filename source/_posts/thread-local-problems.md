---
title: 你有没有被ThreadLocal坑过?
date: 2021-04-12 13:29:53
tags: [Java, ThreadLocal]
---

上一篇文章发出去后，雄哥给我讲写得很好，但是有一些关于ThreadLocal的坑没有指出来。大佬不愧是大佬，一语中的。


因此这篇来看看ThreadLocal存在什么问题，又有怎样的解决方案

<!--more-->

### ThreadLocal的问题

ThreadLocal是线程变量，每个线程都会有一个ThreadLocal副本。每个Thread都维护着一个ThreadLocalMap, 
ThreadLocalMap 中存在一个弱引用Entry。如果一个ThreadLocal没有外部强引用来引用它，那么系统 GC 的时候，这个ThreadLocal势必会被回收。

这样一来，ThreadLocalMap中就会出现key为null的Entry，就没有办法访问这些key为null的Entry的value，如果当前线程迟迟不结束的话，这些key为null的Entry的value就会一直存在一条强引用链
`Thread Ref -> Thread -> ThreaLocalMap -> Entry -> value` 永远无法回收，造成内存泄漏。

虽然在调用get()、set()、remove()的时候会清除所有key为null的value,但是如果调用者不调用也没办法。关于ThreadLocalMap的分析可以参考我写的这篇[开放寻址的ThreadLocalMap分析](https://juejin.cn/post/6844903904933576711/)

![ThreadLocal](/images/java/thread-local.png)

使用ThreadLocal,在子线程中我们是获取不到父线程中的ThreadLocal的值的。

![test-single-thread](/images/java/test-single-thread.png)

输出结果如下

![test-result](/images/java/single-thread-test-result.png)


### 如何获取父线程中ThreadLocal的值？

使用InheritableThreadLocal。InheritableThreadLocal重写了getMap与createMap方法，ThreadLocal使用的是Thread.threadLocals,而InheritableThreadLocal使用的是Thread.inheritableThreadLocals。

![InheritableThreadLocal](/images/java/inherit-thread-local.png)


**问: InheritableThreadLocal是如何实现在子线程中获取父线程中ThreadLocal值？**
答：将父线程中的所有值Copy到子线程


InheritableThreadLocal在创建线程的时候做了一些工作

![创建](/images/java/inherit-create.png)


若父线程inheritableThreadLocals不为null，则为当前线程创建inheritableThreadLocals，并且copy父线程inheritableThreadLocals中的内容,createInheritedMap会创建并拷贝。

总结:

+ 创建InheritableThreadLocal对象时，赋值给了Thread.inheritableThreadLocals变量
+ 创建新的子线程会check父线程的inheritableThreadLocals是否为null, 不为null拷贝父线程inheritableThreadLocals中的内容到当前线程
+ InheritableThreadLocal重写了getMap, createMap, 使用的都是Thread.inheritableThreadLocals变量


### InheritableThreadLocal的问题

在使用线程池的时候InheritableThreadLocal并不能解决获取父线程值得问题，因为线程池中的线程是复用的，可能在子线程中对值进行了修改，使子线程获取到的值并不正确。

![test](/images/java/test-inherit-thread.png)


输出结果如下

```
[main]: aaa

[pool-1-thread-1]: aaa
[pool-1-thread-1]: bbb

[pool-1-thread-2]: aaa
[pool-1-thread-1]: bbb
[pool-1-thread-4]: aaa
[pool-1-thread-1]: bbb
[pool-1-thread-2]: aaa
[pool-1-thread-3]: aaa
[pool-1-thread-2]: aaa
[pool-1-thread-1]: bbb
[pool-1-thread-4]: aaa
[pool-1-thread-3]: aaa

[main]: aaa
```

几个典型应用场景

+ 分布式跟踪系统
+ 日志收集记录系统上下文
+ 应用容器或上层框架跨应用代码给下层SDK传递信息

比如我们的[这样记日志，再也不背锅](https://generalthink.github.io/2021/04/08/record-all-links-log/) 其中的MDC就是使用的InheritableThreadLocal。

解决办法:
1. 线程使用完成，清空TheadLocalMap
2. submit提交新任务时，重新拷贝父线程中的所有Entry。重新为当前线程的inheritableThreadLocals赋值。


### 使用alibab TransmittableThreadLocal

TransmittableThreadLocal就采用了备份的方法来解决这个问题

![TTL](/images/java/ttl-thread-local-test.png)


输出结果

```
[main]: aaa

[pool-1-thread-1]:aaa
[pool-1-thread-1]:bbb

[pool-1-thread-2]:aaa
[pool-1-thread-1]:aaa
[pool-1-thread-3]:aaa
[pool-1-thread-4]:aaa
[pool-1-thread-3]:aaa
[pool-1-thread-1]:aaa
[pool-1-thread-3]:aaa
[pool-1-thread-2]:aaa
[pool-1-thread-1]:aaa
[pool-1-thread-4]:aaa

[main]: aaa
```

TransmittableThreadLocal原理很容易理解，就是在业务逻辑之前先将ThreadLocal中的内容备份，业务逻辑完成后在将内容还原。

![TTL](/images/java/ttl-thread-local.png)


具体的可以参考官方这篇文档: https://github.com/alibaba/transmittable-thread-local/blob/master/docs/developer-guide.md


