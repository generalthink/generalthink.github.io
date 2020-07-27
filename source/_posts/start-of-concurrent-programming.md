---
title: 并发编程不得不考虑的三个问题
date: 2020-06-02 13:46:36
tags: Java
---

编写正确的程序难,编写正确的并发程序则是难上加难。既然这么难为什么还要并发,单线程执行不好吗？为了快呀,点个链接你愿意等1分钟吗?,别说等一分钟了,要是有个网页让我等超过10秒钟,我就马上要关掉了。


我们编写的代码在计算机中运行,那么它肯定会用到计算机中的资源,一般都逃不过cpu、内存以及I/O(文件I/O或者网络I/O等)。但是这三者速度上有极大的差异。

CPU的速度远远快于内存,而内存的速度又远远远快于I/O。

> 比喻: CPU速度相当于 火箭,内存速度相当于 高铁,I/O速度相当于 步行。

而我们的程序运行的快慢实际上是取决于最慢的那个操作--I/O操作,仿佛在这个时候CPU再快都没啥作用。

> 我们一般都说尽可能少的查询数据库(batch的方式更好),就是为了较少I/O操作

<!--more-->

为了合理使用CPU性能,平衡这三者间的速度差。计算机体系结果、操作系统、编译程序都做出了贡献,主要体现在：

1. CPU 增加了缓存,以均衡与内存的速度差异
2. 操作系统增加了进程、线程,以分时复用 CPU,进而均衡 CPU 与 I/O 设备的速度差异
3. 编译程序优化指令执行次序,使得缓存能够得到更加合理地利用

### 缓存导致的可见性问题

单核CPU的时候,所有线程操作的都是同一个CPU的缓存,一个线程对另缓存的写,对另一个线程来说一定是可见的。例如在下面的图中,线程 A 和线程 B 都是操作同一个 CPU 里面的缓
存,所以线程 A 更新了变量 V 的值,那么线程 B 之后再访问变量 V,得到的一定是 V 的最新值(线程 A 写过的值)。

![访问同一个CPU Cache](/images/java/access-singele-cpu-cache.png)

**一个线程对共享变量的修改,另外一个线程能够立刻看到,我们称为可见性。**

但是随着多核时代的来临,每颗 CPU 都有自己的缓存,这时 CPU 缓存与内存的数据一致性就没那么容易解决了,当多个线程在不同的 CPU 上执行时,这些线程操作的是不同的 CPU 缓存。比如
下图中,线程 A 操作的是 CPU1 上的缓存,而线程 B 操作的是 CPU2 上的缓存,很明显,这个时候线程 A 对变量 V 的操作对于线程 B 而言就不具备可见性了

![访问不同CPU Cache](/images/java/access-different-cpu-cache.png)

```java
public class Counter {

  int v = 0;

  public void add() {
      for(int i = 0; i < 10000; i++) {
          v += 1;
      }
  }


  public static void main(String[] args) throws InterruptedException {
      Counter c = new Counter();

      Thread t1 = new Thread(() -> {
          c.add();
      });

      Thread t2 = new Thread(() -> {
          c.add();
      });

      // 启动线程
      t1.start();
      t2.start();

      // 等待两个线程执行结束
      t1.join();
      t2.join();

      System.out.println(c.v);
  }
}

```

比如上面的代码,每次执行的结果都不一样,执行结果也是介于10000和20000之间。

CPU cache中的值什么时候刷新到内存(主存)中是不确定的,所以有可能某个后启动的线程读取到的值不一定是1,而是其他值(代码所示的两个线程启动是存在时间差的)。

### 线程切换带来的原子性问题

你可知道电脑中的进程是交替运行的,你能一边听歌一边看电影都归功于这个进程切换。操作系统允许某个进程执行一小段时间,例如 50 毫秒,过了 50 毫秒操作系统就会重新选
择一个进程来执行(我们称为“任务切换”),这个 50 毫秒称为“时间片”。

![cpu switch](/images/java/cpu-switch.png)

Java 并发程序都是基于多线程的,自然也会涉及到任务切换,也许你想不到,任务切换竟然也是并发编程里诡异 Bug 的源头之一。任务切换的时机大多数是在时间片结束的时候,
我们现在基本都使用高级语言编程,高级语言里一条语句往往需要多条 CPU 指令完成,例如上面代码中的v += 1,至少需要三条 CPU 指令。

1. 指令 1：首先,需要把变量 v 从内存加载到 CPU 的寄存器；
2. 指令 2：之后,在寄存器中执行 +1 操作；
3. 指令 3：最后,将结果写入内存(缓存机制导致可能写入的是 CPU 缓存而不是内存)。

操作系统做任务切换,可以发生在任何一条CPU 指令执行完,是的,是 CPU 指令,而不是高级语言里的一条语句。对于上面的三条指令来说,我们假设 v=0,如果线程 A 在
指令 1 执行完后做线程切换,线程 A 和线程 B 按照下图的序列执行,那么我们会发现两个线程都执行了 v+=1 的操作,但是得到的结果不是我们期望的 2,而是 1。


### 编译优化(指令重排)带来的有序性问题

我们都知道编译器为了优化性能,是会调整语句顺序的。比如下面的代码

```java

int a = 1;

long b = 2L;
```

编译器优化之后可能会变成

```
long b = 2L;
int a = 1;
```

虽然优化后不影响执行结果,不过有时候编译器以及解释器的优化会带来意想不到的结果。

还记得java中获取单例对象的双重检查吗?

```java
public class Singleton {
  static Singleton instance;
  static Singleton getInstance() {
    if(instance == null) {
      synchronized(Singleton.class) {
        if(instance == null)
          instance = new Singleton();
        }
    }
      return instance;
  }
}

```

实际上不能保证上面的代码有效,当我们通过返回的Singleton对象访问其成员变量,就有可能触发空指针异常。
`instance = new Singleton();` 不是原子操作,它由分配空间,初始化对象的字段以及为instance分配地址的多条指令组成。

1. 分配一块内存 M
2. 在内存 M 上初始化 Singleton 对象
3. 然后 M 的地址赋值给 instance 变量。


为了显示实际发生的情况,我使用一些伪代码扩展`instance = new Singleton();`并内联对象初始化代码。

```java
public class Singleton {
  static Singleton instance;
  static Singleton getInstance() {
    if(instance == null) {
      synchronized(Singleton.class) {
        if(instance == null)
          pointer = allocate();
          pointer.field1 = initField1();
          pointer.field2 = initField2();
          instance = pointer;
        }
    }
      return instance;
  }
}

```

为了提高整体性能,某些编译器,内存系统或处理器可能会对指令进行重新排序,例如在初始化对象的字段之前移动 instance = pointer。那么代码就会变成下面这样

```java
public class Singleton {
  static Singleton instance;
  static Singleton getInstance() {
    if(instance == null) {
      synchronized(Singleton.class) {
        if (instance == null)
          pointer = allocate();
          instance = pointer;
          pointer.field1 = initField1();
          pointer.field2 = initField2();
        }
    }
      return instance;
  }
}

```

**这种重新排序是合法的,因为instance = pointer;与初始化字段的指令之间没有数据依赖性。**
但是,这种重新排序(以某些执行顺序)可能导致其他线程看到instance的非null值,但访问了该对象的未初始化字段就会出错。

![重排序导致双重检查失效](/images/java/broken-double-check.png)

**不过如果将instance变量添加上volatile关键字就可以禁止编译优化,就不会出现我们上面所说的问题了。**

### 写到最后

只要在写代码的时候充分考虑上面说的三种情况,那么一定可以帮助你抽丝剥茧的排查多线程下遇到的问题。

巨人肩膀: **极客时间--<java并发编程实战>**

### 关于作者

一个简单的程序员,如果我的写得内容对你有帮助,可以关注公众号: think123

![think123](/images/gzh.png)

