---
title: 万字长文:操作系统如何解决并发问题?
date: 2020-07-02 16:16:11
tags: 操作系统
---



在之前的文章中我们讲到过引起多线程bug的三大源头问题(可见性,原子性,有序性问题),java的内存模型(Java Memory Model)可以解决可见性和有序性问题,但是原子性问题如何解决呢？


```java
public class Counter {

  volatile int count;

  public void add10K() {
    for(int i = 0; i < 10000; i++) {
        count++;
    }
  }

  public int get() {
      return count;
  }

  public static void main(String[] args) throws InterruptedException {

    Counter counter = new Counter();

    Thread t1 = new Thread(() -> counter.add10K());

    Thread t2 = new Thread(() -> counter.add10K());

    t1.start();
    t2.start();

    t1.join();
    t2.join();

    System.out.println(counter.count);
  }

}
```

<!--more-->

上面的代码我们的期望结果是20000,可以运行结果却是小于20000的。是因为count这个共享变量同时被两个线程在修改,而count++这条语句在cpu中实际对应三条指令

1. 读取count的值
2. count + 1
3. 将count+1的值重新复制给count

而线程切换就可能发生在执行完任意一个指令的时候,比如 现在count的值为100, 线程1读取到了之后加1完成,但是还没有将新的值赋值给count,此时让出cpu,然后线程2执行,线程2读取到的值也为100,执行后count++之后,才让出cpu,此时线程1获取到执行资格,将count设置为101。


类似上面的情况,**两个或多个线程读写某些共享数据,而最后的结果取决于线程运行的精准时序,称为竞争条件。**
那么如何避免竞争条件呢？实际上凡是涉及共享内存、共享文件以及共享任何资源的情况都会引发与前面类似的错误,要避免这种错误,关键是要找出某种途径来阻止多个线程同时读写共享的数据。

换言之,我们需要的是**互斥,即以某种手段确保当一个进程在使用另一个共享变量或文件时,其他进程不能做同样的操作。**

上面例子的症结在于,线程1对共享变量的使用未结束之前线程2就使用它了。

线程的存在的问题,进程也同样存在。所以我们接下来看看操作系统是如何处理的。


一个进程的一部分时间做内部计算或另外一些不会引发竞争条件的操作。在某些时候进程可能需要访问共享内存或文件,或执行另外一些会导致竞争的操作。

我们把对共享内存进行访问的程序片段称作临界区域或临界区。如果我们能够使得两个进程不同时处于临界区中,就能避免竞争。

为了保证使用共享数据的并发进程能够正确和高效地进行写作,一个好的解决方案需要满足以下4个条件。

1. 任何两个进程不能同时处于其临界区
2. 不应对CPU的速度和数量做任何假设
3. 临界区以外运行的进程不得阻塞其他进程
4. 不得使进程无限期等待进入临界区

![临界区](/images/java/critical-region.png)

比较期望的进程行为如上图所示。进程A在T1时刻进入临界区。稍后,在T2时刻进程B试图进入临界区,但是失败了,因为另一个进程已经在临界区内,而一个时刻只允许一个进程在临界区内。所以B被暂时挂起知道T3时刻A离开临界区为止,从而允许B立即进入。最后,在B离开(在时刻T4),回到了在临界区中没有进程的原始状态。

### 互斥

为了保证同一时刻只能有一个进程进入临界区,我们需要让进程在临界区保持互斥,这样当一个进程在临界区更新共享内存时,其他进程将不会进入临界区,以免带来不可预见的问题。

接下来会讨论下关于互斥的几种方案。

#### 屏蔽中断

单处理器系统中比较简单的做法就是在进入临界区之后立即屏蔽所有中断,并在要离开之前在打开中断。屏蔽中断后,时钟中断也被屏蔽,由于cpu只有发生时钟中断或其他中断时才会进行进程切换,这样在屏蔽中断后CPU不会被切换到其他进程,检查和修改共享内存的时候就不用担心其他进程介入。

> 中断信号导致CPU停止当前正在做的工作并且开始做其他的事情

可是如果把屏蔽中断的能力交给进程,如果某个进程屏蔽之后不再打开中断怎么办？
整个系统可能会因此终止。多核CPU可能要好点,可能只有其中一个cpu收到影响,其他cpu仍然能正常工作。


#### 锁变量

锁变量实际是一种软件解决方案,设想存在一个共享锁变量,初始值为0,当进程想要进入临界区的时候,会执行下面代码类似的判断

```

if( lock == 0) {
  lock = 1;

  // 临界区
  critical_region();
} else {
   wait lock == 0;
}

```

看上去很好,可是会出现和文章开头演示代码同样的问题。进程A读取锁变量lock的值发现为0,而在恰好设置为1之前,另一个进程被调度运行,将锁变量设置为1。
当第一个进程再次运行时,同样也将该锁设置为1,则此时同时有两个进程进入临界区中。

同样的即使在改变值之前再检查一次也是无济于事,如果第二个进程恰好在第一个进程完成第二次检查之后修改了锁变量的值,则同样会发生竞争条件。

#### 严格轮换法

这种方法设计到两个线程执行不同的方法

线程1:

```java

while(true) {
  
  while(turn != 0) {
    // 空等
  }
  // 临界区
  critical_region();
  
  turn = 1;
  // 非临界区
  noncritical_region();
}


```

线程2:

```java
while(true) {
  
  while(turn != 1) {
    // 空等
  }
  // 临界区
  critical_region();
  
  turn = 0;

  noncritical_region();
}

```
turn初始值为0,用于记录轮到哪个线程进入临界区。

开始时,进程1检查turn,发现其值为0,于是进入临界区。进程2也发现值为0,于是一直在一个等待循环中不停的测试turn的值。

**连续测试一个变量直到某个值出现为止,称为忙等待。由于这种方式浪费CPU时间,所以通常应该避免。只有在有理由认为等待时间是非常短的情况下,才使用忙等待。用于忙等待的锁,称为自旋锁。**


看上去是不是很完美,其实不然。

当进程2执行得比较慢还在执行非临界区代码,此时turn = 1,而进程1又回到了循环的开始,但是它不能进入临界区,它只能在while循环中一直忙等待。


这种情况违反了前面叙述的条件3：进程不能被一个临界区外的进程阻塞。这也说明了在一个进程比另一个慢了许多的情况下,轮流进入临界区并不是一个好办法。


#### Peterson算法

1981年发明的互斥算法其基本原理如下

```java
int turn;
boolean[] interested = new boolean[2];

// 进程号是0或者1
void enterRegion(int process) {
  
  // 其他进程
  int other = 1 - process;

  // 记录哪个进程有兴趣进入临界区
  interested[process] = true;

  // 设置标志
  turn = process;
  
  // 如果不满足进入条件,则空等待
  while(turn == process && interested[other] == true) {

  }
}

void leaveRegion(int process) {
  interested[process] = false;
}
```

当进程想要进入临界区的时候就会调用enterRegion方法,该方法在需要时将使得进程等待,在完成对共享变量的操作后,就会调用leaveRegion离开临界区。

这种算法虽然不会严格要求进程的执行顺序,但是仍然会造成忙等待。

### 睡眠与唤醒

忙等待的缺点如何克服呢？
> 忙等待的本质是由于当一个进程想要进入临界区时,先检查是否允许进入,如不允许,则该进程在原地一直等待,知道允许为止

是否存在一个命令当无法进入临界区的时候就阻塞,而不是忙等待呢？

进程通信有两个原语sleep和wakeup就满足这个需求,sleep是一个将引起调用进程阻塞的系统调用,它会使得进程被挂起直到另外一个进程将其唤醒。

而wakeup有一个参数,即被需要唤醒的线程。这里我们来考虑一个生产者消费者的例子。

生产者往缓冲区中写入数据,消费者从缓冲区中取出数据。但是这里有两个隐藏的条件,当缓冲区为空的时候消费者不能取出数据,
当缓冲区满了的时候生产者不能生产数据。

```java
int MAX = 100;
int count = 0;

void producer() {
  
  int item;

  while(true) {
    iterm = produce_item();
    // 队列满了就要睡眠
    if(count == MAX) {
      sleep();
    }

    insert_item(item);

    count = count + 1;

    if(count == 1) {
      wakeup(consumer);
    }

  }
}

void consumer() {
  int item;

  while(true) {
    if(count == 0) {
      sleep();
    }

    item = remove_item();

    count = count -1;

    // 队列不满,则唤醒producer
    if(count == N -1) {
      wakeup(producer);
    }

    consume_item(item);
  }
}

```

乍看之下没有问题,其实还是存在问题的,那就是对count(队列当前数据个数)的访问未加限制。

队列为空,当consumer在执行到if(count == 0)的时候,如果刚好发生了进程切换,producer执行,此时插入一个数据,然后发现count == 1,就会唤醒consumer(但是实际上consumer并没有睡眠),consumer接着刚才的代码执行会发现count = 0,于是睡眠。生产者迟早会填满整个队列,从而使得两个进程都陷入睡眠状态。

快速的弥补方法就是加一个唤醒等待位,当一个wakeup信号发送给一个清醒的进程时,将该标志位置为1,随后,当该进程要睡眠时,如果唤醒等待位为1,则将该标志位清除,同时进程保持为清醒状态。

但是这个办法还是治标不治本,如果有三个进程甚至更多的进程那么就会需要更多的标志位。

### 信号量

信号量是E.W.Dijkstra在1965年提出的一种方法,它使用一个整型变量来累计唤醒次数,供以后使用。
在他的建议中引入了一个新的变量类型,称作信号量(semaphore),一个信号量的取值可以为0(表示没有唤醒操作)或者为正值(表示有一个或多个唤醒操作)。
Dijkstra建议设立两种操作:down和up(分别为一般化后的sleep和wakeup)

```java

void down() {
  if(semaphore > 0) {
      semaphore = semaphore -1;
  } else {
      sleep();
  }
}
```

**检査数值、修改变量值以及可能发生的睡眠操作均作为一个单一的、不可分割的原子操作完成,保证一旦一个信号量操作开始,则在该操作完成或阻塞之前,其他进程均不允许访问该信号量。**
> 所谓原子操作,是指一组相关联的操作要么都不间断地执行,要么都不执行。

```java
void up() {
  
  semaphore = semaphore + 1;

  wakeup();
}

```
up操作对信号量的操作增加1,同样信号量的值增加1和唤醒一个进程同样是不可分割的一个操作。


那么是如何保证down()和up()的操作是原子性的呢？实际上是通过硬件提供的TSL或XCHG指令来完成的。

> 执行TSL或者XCHG指令的CPU将锁住内存总线,以禁止其他CPU在本指令结束之前访问内存。


来看看如何使用信号量的方式来解决生产者消费者问题。


在下面的实现方案中使用了3个信号量

1. full: 记录充满的缓冲槽数目
2. empty: 记录空的缓冲槽数目
3. mutex: 供两个或多个进程使用的信号量,初始值为1,表示只有一个进程可以进入临界区

```c


#define N 100  // 缓冲区中槽数目

typedef int semaphore; // 信号量是一种特殊的整型数据

semaphore mutex = 1; // 控制对临界区的访问
semaphore empty = N; //缓冲区的空槽数目
semaphore full = 0; // 缓冲区的满槽数目
void producer(void) {
  int item;
  while (TRUE) {
    
    // 产生放在缓冲区中的一些数据
    item = produce_item();
    // 将空槽数目減1
    down(&empty);
    // 进入临界区
    down(&mutex);

    // 将新数据项放到缓冲区中
    insert_item(item);
    // 离开临界区
    up(&mutex);

    // 将满槽的数目加1
    up(&full);
  }
}

void consumer(void) {
  
  int item;
  while (TRUE) {
    // 将满槽数目減1
    down(&full);
    // 进入临界区
    down(&mutex);
    // 从缓冲区中取出数据项
    item = remove_item();
    // 离开临界区
    up(&mutex);
    // 将空槽数目加1
    up(&empty);
    // 处理数据项
    consume_item(item);
  }
}
```

empty这个信号量保证了缓冲区满(empty=0)的时候生产者停止运行,full这个信号量保证了缓冲区空的时候消费者停止运行。

### 互斥量

互斥量是信号量的简化版本,互斥量仅仅适用于管理共享资源或一小段代码,由于互斥量在实现时即容易又有效,这使得互斥量在实现用户空间线程包时非常有用。

互斥量是一个可以处于两态之一的变量:解锁和加锁。这样,只需要一个二进制位表示它,不过实际上,常常使用一个整型量,0表示解锁,而其他所有的值则表示加锁。

当一个线程(或进程)需要访问临界区时,它调用mutex_lock,如果该互斥量当前是解锁的(即临界区可用),此调用成功,调用线程可以自由进入该临界区。

另一方面,如果该互斥量已经加锁,调用线程被阻塞,直到在临界区中的线程完成并调用mutex_unlock。如果多个线程被阻塞在该互斥量上,将随机选择一个线程并允许它获得锁。
由于互斥量非常简单,所以如果有可用的TSL或XCHG指令,就可以很容易地在用户空间中实现它们。

```
mutex_lock:
  // 将互斥信号量复制到寄存器,并且将互斥信号量置为 1
  TSL REGISTER,MUTEX
  // 互斥信号是0吗
  CMP REGISTER,#O
  // 如果互斥信号量为0,它被解锁,所以返回
  JZE ok
  // 互斥信号量忙,调度另一个线程 
  CALL thread_yield
  // 稍后再试 
  JMP mutex_lock
ok: RET  // 返回调用者,进入临界区

mutex_unlock:
  // 将mutex置为0 
  MOVE MUTEX,#0
  // 返回调用者
  RET 1
```

mutex_lock的方法和enter_region的方法有点类似,但是有一个很关键的区别,enter_region进入临界区失败,会忙等待(实际上由于CPU时钟超时,会调度其他进程运行)。


由于thread_yield只是在用户空间中对线程调度程序的一个调用,所以它的运行非常快。这样,mutex_lock及mutex_unlock都不需要任何内核调用。


#### 线程中的互斥

为实现可移植的线程程序,IEEE标准1003.1c中定义了线程的标准,它定义的线程包叫做Pthread,大部分UNIX系统都支持这个标准。

Pthread提供许多可以用来同步线程的函数。其基本机制是使用一个可以被锁定和解锁的互斥量来保护每个临界区。

与互斥量相关的主要函数调用如下所示：

| 线程调用  | 描 述  |
|---|---|
| pthread_mutex_init  | 创建一个互斥量  |
| pthread_mutex destroy  | 撤销一个已存在的互斥量  |
| pthread_mutex_lock  | 获得一个锁或阻塞 |
| pthread_mutex_trylock  | 尝试获得一个锁或失败  |
| pthread_mutex_unlock  | 释放一个锁  |

 
 

除互斥量之外,pthread提供了另一种同步机制:条件变量。

**互斥量在允许或阻塞对临界区的访问上是很有用的,条件变量则允许线程由于一些未达到的条件而阻塞,绝大都分情況下这两种方法是一起使用的。**现在让我们进一步地研究线程、互斥量、条件变量之间的关联。

再考虑一下生产者一消费者问题:一个线程将产品放在一个缓冲区内,而另一个线程将它们取出。如果生产者发现缓冲区中没有空槽可以使用了,它不得不阻塞起来直到有一个空槽可以使用。生产者使用互斥量可以进行原子性检查,而不受其他线程干扰。但是当发现缓存区已经满了以后,生产者需要一种方法来阻塞自己并在以后被唤醒,这便是条件变量做的事了.

与条件变量相关的pthread调用如下:

| 线程调用  | 描 述  |
|---|---|
| pthread_condinit  | 创建一个条件变量  |
| pthread_cond_destroy  | 撤销一个条件变量  |
| pthread_cond_wait  | 阻塞以等待一个信号  |
| pthread_cond_signal  | 向另一个线程发信号来唤醒它  |
| pthread_cond_broadcast  | 向多个线程发信号来让它们全部唤醒  |


与条件变量相关的最重要的两个操作是pthread_cond_wait 和 pthread_cond_signal,前者阻塞调用线程直到其他线程给它发信号(pthread_cond_signal通知其他进程)。当有多个线程被阻塞在同一个信号时,可以使用pthread_cond_broadcast 调用。

**条件变量与互斥量经常一起使用,这种模式用于让一个线程锁住一个互斥量,然后当它不能获得它期待的结果时等待一个条件变量。最后另一个线程会向它发信号,使它可以继续执行。
pthread_cond_wait原子性地调用并解锁它持有的互斥量,由于这个原因,互斥量是参数之一。**

```c
#include <stdio.h>
#include <pthread.h>
#define MAX 1000000000
pthread_mutex_t the_mutex;
pthread_cond_t condc, condp;

// 缓冲区
int buffer = 0;
void producer(void *ptr){
  int i;
  for (i =1; i <= MAX; i++) {
    // 互斥使用缓冲区
    pthread_mutex_lock(&the_mutex);
    while (buffer != 0) {
      pthread_cond_wait(&condp, &the_mutex);
    }
    // 将数据放入缓冲区
    buffer = i;
    // 唤醒消费者
    pthread_cond_signal(&condc);
    // 释放缓冲区
    pthread_mutex_unlock(&the_mutex);
  }

  pthread _exit(O);
}

void consumer(void *ptr) {
  int i;
  for (i = 1; i <= MAX; i++) {
    // 互斥使用缓冲区
    pthread_mutex_lock(&the_mutex);
    while (buffer == 0) {
       pthread_cond_wait(&condc, &the_mutex);
    }
    // 从缓冲区中取出数据
    buffer = 0;
    // 唤醒生产者
    pthread_cond_signal(&condp);
    // 释放缓冲区
    pthread_.mutex_unlock(&the_mutex);
  }
  pthread_exit(O);
}


int main(int argc, char** argv) {

  // 初始化
  pthread_t pro, con;
  pthread_mutex_init(&the_mutex, 0);
  pthread_cond _init(&condc, 0);
  pthread_cond_init(&condp, 0);
  
  // 创建一个线程(pthread提供的方法)
  pthread_create(&con, 0, consumer, 0);
  pthread_create(&pro, 0, producer, 0);

  // 等待线程执行完成
  pthread_join(pro, 0);
  pthread_join(con, 0);

  pthread_cond_destroy(&condc);
  pthread_cond_destroy(&condp);
  pthread_mutex_destroy(&the_mutex); 
}

```

这个例子虽然简单,但是却说明了基本的机制。
**同时使一个线程睡眠的语句应该总要检查这个条件,以保证线程在继续执行前满足条件,因为线程可能已经因为一个UNIX信号或其他原因被唤醒。**

### 管程

有了信号量和互斥量之后,进程间通信看起来就容易了,是这样吗？

当然不是,可以再看看信号量中的producer(),如果将代码中的两个down操作交换下次序,这使得mutex的值在empty之前减1,如果缓冲区完全满了,生产者将阻塞,此时mutex为0,

当消费者下一次访问缓冲区时,消费者也就被阻塞,这种情况叫做死锁。

```java
void producer(void) {
  int item;
  while (TRUE) {
    
    // 产生放在缓冲区中的一些数据
    item = produce_item();

    // 下面的代码交换了顺序

    // 进入临界区
    down(&mutex);
    // 将空槽数目減1
    down(&empty);
   
    // 将新数据项放到缓冲区中
    insert_item(item);
    // 离开临界区
    up(&mutex);

    // 将满槽的数目加1
    up(&full);
  }
}
```

这个问题告诉我们使用信号量要很小心。

为了更易于编写正确的程序,Brinch Hansen(1973)和Hoare(1974)提出了一种高级同步原语,称为管程(monitor)。
在下面的介绍中我们会发现,他们两人提出的方案略有不同。

**一个管程是一个由过程、变量及数据结构等组成的一个集合,它们组成一个特殊的模块或软件包。**

下面的代码是类Pascal语法,procedure你可以认为是一个函数。

```
monitor example:
 interger i;
 condition c;
 procedure producer();
 begin
 ...
 end;

 procedure consumer();
 begin
 ...
 end;
end monitor;
```
管程有一个很重要的特性,即任一时刻管程中只能有一个活跃进程,这一特性使管程能有效地完成互斥。

> 管程是一个语言概念

管程是编程语言的组成部分,编译器知道它们的特殊性,因此可以采用与其他过程调用不同的方法来处理对管程的调用。

**当一个进程调用管程过程时,该过程中的前几条指令将检査在管程中是否有其他的活跃进程。如果有,调用进程将被挂起,直到另一个进程离开管程将其唤醒。如果没有活跃进程在使用管程,则该调用进程可以进入。**

进入管程时的互斥由编译器负责,但通常的做法是用一个互斥量或信号量。因为是由编译器而非程序员来安排互斥,所以出错的可能性要小得多。在任一时刻,写管程的人无须关心编译器是如何实现互斥的。他只需知道将所有的临界区转换成管程过程即可,决不会有两个进程同时执行临界区中的代码。


**管程仍然需要一种方法使得进程在无法继续运行时被阻塞,其解决办法就是条件变量,以及相关的两个操作:wait和signal.**

当一个管程发现他无法运行时(比如生产者无法缓冲区满了),它会在某个条件变量执行wait操作,该操作导致调用进程自身阻塞,并且还将另一个以前在管程外的进程调入管程,另一个进程比如消费者可以唤醒生产者,通过对生产者正在等待的条件变量执行signal。

#### signal唤醒策略

为了避免管程中同时有两个活跃进程,我们需要一个规则来通过在signal之后怎么做。

Hoare 建议让唤起的新线程执行,而挂起另一个线程。

Brinch Hansen则建议执行signal的进程必须立即退出管程,即signal语句只能作为一个管程过程的最后一个语句。

第三种方法是让发信号者继续运行,并且只有在发信号者退出管程后,才允许等待的进程开始运行。而java使用的就是该种方案。


需要注意的是无论是信号量还是管程在分布式系统中都是不起作用的,至于为什么我相信大家心里肯定有了答案。

