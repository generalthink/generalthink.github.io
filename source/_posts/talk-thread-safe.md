title: 浅谈java线程安全
date: 2016-09-21 22:27:34
tags: java
categories: java
description:
keywords: java多线程 volitale synconized ThreadLocale
---
### 进程与线程
计算机的核心是CPU，它承担了所有的计算任务。它就像一座工厂，时刻在运行。
假定工厂的电力有限，一次只能供给一个车间使用。也就是说，一个车间开工的时候，其他车间都必须停工。背后的含义就是，单个CPU一次只能运行一个任务。
进程就好比工厂的车间，它代表CPU所能处理的单个任务。任一时刻，CPU总是运行一个进程，其他进程处于非运行状态。
一个车间里，可以有很多工人。他们协同完成一个任务。
线程就好比车间里的工人。一个进程可以包括多个线程。

### JAVA中的多线程
一般来说，当运行一个应用程序的时候，就启动了一个进程，当然有些会启动多个进程。启动进程的时候，操作系统会为进程分配资源，其中最主要的资源是内存空间，因为程序是在内存中运行的。在进程中，有些程序流程块是可以乱序执行的，并且这个代码块可以同时被多次执行。实际上，这样的代码块就是线程体。线程是进程中乱序执行的代码流程。当多个线程同时运行的时候，这样的执行模式成为并发执行。
多线程的目的是为了最大限度的利用CPU资源。
 
Java编写程序都运行在在Java虚拟机（JVM）中，在JVM的内部，程序的多任务是通过线程来实现的。每用java命令启动一个java应用程序，就会启动一个JVM进程。在同一个JVM进程中，有且只有一个进程，就是它自己。在这个JVM环境中，所有程序代码的运行都是以线程来运行。
一般常见的Java应用程序都是单线程的。比如，用java命令运行一个最简单的HelloWorld的Java应用程序时，就启动了一个JVM进程，JVM找到程序程序的入口点main()，然后运行main()方法，这样就产生了一个线程，这个线程称之为主线程。当main方法结束后，主线程运行完成。JVM进程也随即退出 。

当我们在tomcat服务器中运行一个WEB应用程序的时候，其实就是启动了一个JAVA进程，对WEB界面的每个操作都是在一个单独的线程中执行的，对于相同的操作每个线程执行的方法体是一样的(比如说同时有多个人在淘宝查看同一个商品的详细)。

### Java中如何使用多线程
1. 直接继承java.lang.Thread或者直接使用java.lang.Thread
```java
    new Thread().start();
```
2. 实现java.lang.Runnable接口,并将其作为参数传递给java.lang.Thread
```java
 new Thread(new Runnable() {
      
      @Override
      public void run() {
        
      }
    }).start();
```
注意，这里调用的是start方法，只有调用start方法才会启动线程，然后线程会执行run方法中的代码,如果直接调用run方法,java只是当成一个普通的方法调用，并不会启动一个线程
### 多线程的使用场景
刚才也说了线程是为了更好的利用CPU资源,那么我们一般在哪些场景下使用多线程呢？当然这并没有一个统一的答案，我在这里总结一下我使用多线程的场景
1. 在数据库备份等比较耗时的操作中使用多线程,后台默默执行即可
2. 系统需要定时做一些任务的时候，比如定时发送邮件,更新配置等
3. 需要异步处理的数据的时候,比如说前台触发完成一个任务,后台线程直接执行，前台只需要轮询状态即可。

### 什么是线程安全
线程安全指的是多线程环境下,一个类在在执行某个方法的时候,对类的内部实例变量的访问是安全的。因此对于下面列出来的2类变量,并不存在任何线程安全的说法：
1. 方法签名中的任何参数对象
2. 处于方法内部的局部变量

**为什么呢？**
Java虚拟机在运行java程序的过程中会将它所管理的内存区域划分成为几个不同的区块,其将会包括以下几个运行时数据区块
![Java运行时数据区](/images/thread-java-running-data-area.png)

从上图中可以看出有所有线程共享的数据库只有堆和方法区,方法区用于存取以被虚拟机加载的类信息、常量、静态变量、即时编译器编译后的代码等数据,而堆的唯一目的就是存放对象实例,几乎所有的对象实例都在这里分配内存。
> 一个本地变量可能是原始类型，在这种情况下，它总是“呆在”线程栈上。
> 
> 一个本地变量也可能是指向一个对象的一个引用。在这种情况下，引用（这个本地变量）存放在线程栈上，但是对象本身存放在堆上。

> 一个对象可能包含方法，这些方法可能包含本地变量。这些本地变量任然存放在线程栈上，即使这些方法所属的对象存放在堆上。

> 一个对象的成员变量可能随着这个对象自身存放在堆上。不管这个成员变量是原始类型还是引用类型。

> 静态成员变量跟随着类定义一起也存放在堆上。

> 存放在堆上的对象可以被所有持有对这个对象引用的线程访问。当一个线程可以访问一个对象时，它也可以访问这个对象的成员变量。如果两个线程同时调用同一个对象上的同一个方法，它们将会都访问这个对象的成员变量，但是每一个线程都拥有这个本地变量的私有拷贝。

更多关于java内存模型数据可以参考<http://ifeve.com/java-memory-model-6/>
### 线程不安全的代码
前面也说了,多线程是完成同一件任务的,那么必然存在对资源的竞争,那么在竞争的时候,对资源的不正当使用就会导致程序出现问题，例如下面这段代码
```java
public class MyThread extends Thread{
  protected MySalary mySalary;
  
  protected int value;
  
  public MyThread(MySalary mySalary,int value) {
    this.mySalary = mySalary;
    this.value = value;
  }
  
   @Override
  public void run() {
     mySalary.add(value);
  }
  
  
  public static void main(String[] args) throws InterruptedException {
    MySalary ms = new MySalary();
    MyThread t1 = new MyThread(ms,200);
    MyThread t2 = new MyThread(ms,300);
    t1.start();
    t2.start();
    Thread.sleep(1000);//主线程睡眠1秒钟,保证两个线程运行完毕
     System.out.println(Thread.currentThread().getName() + " run over and salary = " + salary);
  }
  
}

class MySalary {
  public int salary = 1000;
  public void add(int value) {
    try {
      Thread.sleep(150);
      this.salary = salary + value;
      //打印出当前运行的线程
      System.out.println(Thread.currentThread() + " run and salary = " + salary);
    } catch (InterruptedException e) {
      e.printStackTrace();
    }
 }
}

```
预期结果应该打印的是1500,但是实际情况可能与预期结果不符,在本地多运行几次之后出现了这样的结果:
```
Thread-0 run over and salary = 1300
Thread-1 run over and salary = 1300
main run over and salary = 1300
```
这就是多线程运行导致的线程不安全的结果!

### 为什么会导致线程不安全
在没有同步的情况下,Java内存模型允许编译器对操作顺序进行重排序,并将数值缓存在寄存器中,此外,它还允许CPU对操作顺序进行重排序,并将数值缓存在寄存器中。所以在缺乏足够同步的多线程程序中,要想对内存的操作顺序进行判断,几乎无法得出正确的结论。

之前说了Java虚拟机对内存的划分,但是其和硬件是存在差异,在硬件级别上是不存在堆栈的划分的,压根没有这个概念。
![硬件中对于内存的划分](/images/thread-java-memory-model-4.png)

要知道,CPU的运行速度是非常快的,CPU的运行数据是从内存(主存)来的,但是从内存中取数据对于CPU来说是非常慢的,对于一些常用的数据,CPU就做了一些缓存,所以有了CPU的缓存。而我们上面造成的多线程问题也正是由于这种"信息不对等"的情况导致的。刚才也说了硬件和JVM的运行时区并不一样,那么它们之间的关系是怎样的呢?这里同样可以用一张图来描述它们的关系
![JVM与硬件的交互](/images/thread-java-memory-model-5.png)

这个时候就可以解释为什么会出现上面代码运行的结果了
![线程不安全代码解析](/images/thread-java_running_result.png)

### synconized的作用以及使用
synconized是java中的关键字,使用它可以保证线程安全,同步块在Java中是同步在某个对象上。所有同步在一个对象上的同步块在同时只能被一个线程进入并执行操作。所有其他等待进入该同步块的线程将被阻塞，直到执行该同步块中的线程退出。
这就相当于寝室6个同学同时要上厕所,然后一起抢厕所，抢到厕所的那个人会把厕所门给关上,表示厕所有人,然后其他人只能在门外等着,等别人上完厕所了,其他人在接着抢。
那么上面线程不安全的代码可以进行部分改写
```java
 synchronized (this) {
    this.salary = salary + value;
}
```
此时，无论启动多少个线程都能够保证线程的安全了,synchronized不仅可以使用在代码块上还可以使用在方法声明上,但是要注意的是synconized是比较耗费系统资源的,所以要尽可能小范围的使用。

### volitale的使用
volitale的作用是保证内存可见性,如果一个成员变量被valitale修饰,在A线程中修改了valitale变量的值，对其他线程来说该修改是可见的。
即读取valitale变量相当于进入同步代码块,写入valitale变量相当于退出同步代码块,而且读取valitaile变量的开销仅仅只比非valitale变量的开销略高一点,更远远低于synchronized.
值得注意的是volitale并不能保证count++这样操作的原子性,所以volitale并不能保证线程安全。

**因此synchronized和volitale的区别是:加锁机制既可以确定可见性又可以保证原子性,而volatile变量只能确保可见性。**

所以它的使用场景一般包括:确保它们自身状态的可见性,确保它们所引用状态的可见性，以及标示一些重要的程序生命周期的事件的发生(例如初始化或者关闭)。
一种典型的使用方式:检查某个判断标记判断是否退出循环
```java
valitale boolean isExit;
while(!isExit) {
    doSomething();
}
```
这样当isExit标志被另外一个线程修改为true的时候,执行判断的线程就能够准确的读取到isExit的值，从而退出循环。
    
### ThreadLocal的使用
既然提到了Thread,就不得不说ThreadLocal了,先看下ThreadLocal内部实现机制:
```java
    public void set(T value) {
        Thread t = Thread.currentThread();
        //通过当前线程获取当前线程的本地变量ThreadLocalMap
        ThreadLocalMap map = getMap(t);
        if (map != null)
            map.set(this, value);
        else
            createMap(t, value);
    }

    public T get() {
        Thread t = Thread.currentThread();
        ThreadLocalMap map = getMap(t);
        if (map != null) {
            //获取当前线程中以当前ThreadLocal实例为key的变量值
            ThreadLocalMap.Entry e = map.getEntry(this);
            if (e != null)
                return (T)e.value;
        }
        //当map不存在时,设置初始值,可以通过new ThreadLocal时覆盖initialValue方法,这也是setInitialValue主要的调用方法
        return setInitialValue();
    }

    ThreadLocalMap getMap(Thread t) {
        return t.threadLocals;
    }

    void createMap(Thread t, T firstValue) {
        t.threadLocals = new ThreadLocalMap(this, firstValue);
    }
```
从ThreadLocal的内部实现机制分析,synchronized这种机制是提供一份变量让不同的线程排队访问,而ThreadLocal是为每个线程都提供一份变量的副本,从而实现同时访问而不受影响。
从这里也看出来了两者之间的应用场景不同,synchronized是为了让每个线程对变量的修改都对其他线程可见,而ThreadLocal是为了线程对象的数据不受其他线程影响,它最适合的场景应该是在同一线程的不同开发层次中共享数据。

### 安全的发布对象
我们分析了,其实线程不安全就是因为多个线程对公共资源(存在于堆中的对象)的不正当利用,那么如何保证发布的对象线程安全呢？
一个正确构造的对象可以通过以下方式来安全的发布:
1. 在静态初始化函数中初始化一个对象引用(JVM在静态初始化的时候会保证线程安全)
2. 将对象的引用保存到volatile类型的域或者AtomicReferance中
3. 将对象的引用保存到某个正确构造对象的的final类型域中
4. 将对象的引用保存到一个由锁保护的域中

### Java中其他保证线程安全的方式 
1. 通过将一个键或者值放入HashTable、synchronizedMap或者ConcurrentMap中，可以安全的将它发布给任何从这些容器中访问它的线程(无论是直接访问还是通过迭代器访问)
2. 通过将某个元素放入Vector、CopyOnWriteArrayList、CopyOnWriteArraySet、synchronizedList或者synchronizedSet中,可以将元素安全的发布到任何从这些容器中访问它的线程
3. 通过将某个元素放入BlockingQueue或者ConcurrentLinkedQueue中,可以将元素安全的发布到任何从这些队列中访问该元素的线程

类库中的其他机制也可以实现安全发布,比如Futrue和Exchange.

### 写一个线程安全的单例模式
1. 饿汉式
```java
public class MyFactory {

    public static MyFactory factory = new MyFactory();

    private MyFactory() {
    }

    public static MyFactory getFactory() {
        return factory;
    } 
}

```
2. 懒汉式
```java
public class MyFactory {
    //这里的volitale不可或缺
    public volitale static MyFactory factory = null;

    private MyFactory() {
    }

    public static MyFactory getFactory() {
       if(factory == null){
            synchronized (MyFactory.class){
                if(factory == null){
                    factory = new MyFactory();
                }
            }
        }
        return factory;
    } 
}

```

懒汉式使用了"双重检查锁",在上锁之前多进行一次null检查就可以减少绝大多数的加锁操作。当然这种写法也不是唯一的,你可以使用静态内部类,还使用枚举(这种是Effective Java的推荐写法)。使用枚举除了线程安全和防止反射强行调用构造器之外(上面起到的其他方式均不能避免这种情况)，还提供了自动序列化机制，防止反序列化的时候创建新的对象。

### 写在最后

线程安全说简单也简单,说难也难,简单就简单在控制好被线程共享的资源即可,难就难在在控制的同时保证程序运行效率以及不出现线程安全问题。当然这设计到很多方面了，本文也只是作者在了解了多线程之后写下的粗浅见接，这其中也参考了很多资料,如有错误或者遗漏,请指出。

### 参考资料
1. http://www.ruanyifeng.com/blog/2013/04/processes_and_threads.html
2. http://ifeve.com/java-memory-model-6/