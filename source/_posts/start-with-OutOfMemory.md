title: 从OutOfMemory开始
date: 2016-10-31 20:57:56
tags: java
categories: [java,多线程]
description: 我们经常会遇到OutOfMemoryError,我们通常的使用手段是增大堆内存,但是真正导致这个错误的原因有哪些呢？
keywords: java 多线程 OutOfMemory
---

我们来看一下下面这段异常信息
```java
java.lang.OutOfMemoryError: Java heap space
    at test.OutofMemoryTest.heapOutOfMomory(OutofMemoryTest.java:27)
```
看见上面的这段错误信息是不是很熟悉,经常我们在编写代码的过程中会遇到这样的错误,那么引起这个错误的原因有哪些呢？又有哪些类型的OutOfMemory呢？

我们知道JVM运行时数据区域是这样的：
![JVM运行时区域](/images/thread-java-running-data-area.png)
在JVM规范的描述中,除了程序计数器之外,虚拟机内存的其他几个运行时区域都有发生OutOfMemoryError异常的可能。

### Java堆溢出

我们看看导致以下这段导致文章开头那段异常信息的代码
```java
@Test
//VM Args：-Xms10m  -Xmx10m -XX:+HeapDumpOnOutOfMemoryError
  public void heapOutOfMomory() {
    Byte[] b = new Byte[20*1024*1024];
  }
```
代码其实很简单,但是为什么会导致OutOfMemoryError呢？原因是我们设置了堆的大小为10M(通过-Xmx设置堆的最大值,-Xms设置堆的最小值),而我们在代码中却申请了20M的空间,所以会导致抛出OutOfMemoryError。此种问题一般可以通过设置-XX:+HeapDumpOnOutOfMemoryError参数让虚拟机在出现内存溢出异常时Dump出当前的内存堆转储快照以便事后进行分析(当然可以通过java自带的jmap命令以及jstat命令联合查看,也可通过Eclipse Memory Analyzer进行分析)。

### 虚拟机栈和本地方法栈溢出
由于虚拟机并不区分虚拟机栈和本地方法栈,因此对于HotSpot来说,虽然-Xoss参数(设置本地方法栈大小)存在,但是实际上是无效的,栈容量只能由-Xss参数设定。关于虚拟机栈和本地方法栈,在Java虚拟机规范中描述了两种异常：
1. 如果线程请求的的栈深度大于虚拟机所允许的最大深度,将抛出StackOverflowError异常
2. 如果虚拟机在扩展栈时无法申请到足够的内存空间,则抛出OutOfMemoryError异常。

```java
public class OutofMemoryTest {
  private int statckLen = 1;
  
  public void statckLeak() {
    statckLen ++;
    statckLeak();
  }
  public static void main(String[] args) {
    OutofMemoryTest test = new OutofMemoryTest();
    try {
      test.statckLeak();
    } catch (Throwable e) {
      System.out.println("stack length = " + test.statckLen);
      throw e;
    }
  }
```

在运行过程中设置了-Xss128K(如果这个值设置太小就会提示The stack size specified is too small, Specify at least 104k),程序抛出的异常信息为:
```
stack length = 990
Exception in thread "main" java.lang.StackOverflowError
    at test.OutofMemoryTest.statckLeak(OutofMemoryTest.java:41)
```
可以看出这里并没有抛出OutOfMemoryError,很明显,在单线程情况下,无论是栈帧太大还是虚拟机容量太小,当内存无法分配的时候,虚拟机抛出的都是StackOverflowError异常。
如果测试不限于单线程呢？通过不断建立线程的方式倒是可以产生内存溢出异常。
```java
//VM Args：-Xss20M -Xmx7168M -Xms7168M -XX:MaxPermSize1024M -XX:PermSize1024M
  public void statckLeakByThread() {
    while(true) {
        new Thread(new Runnable() {
            
            @Override
            public void run() {
                while(true) {
                }
            }
        }).start();
  }
}
```
运行这段代码极容易使得电脑卡死,需要尤其注意,在我的测试环境中抛出的异常信息如下:
```
java.lang.OutOfMemoryError: unable to create new native thread
    at java.lang.Thread.start0(Native Method)
    at java.lang.Thread.start(Unknown Source)
    at com.think.base.OutOfMemoryTest.stackLeak(OutOfMemoryTest.java:40)
```
但是这样产生的内存溢出异常与栈空间是否足够大并不存在任何联系,或者可以这样说,在这种情况下,为每个线程的栈分配的内存越大,反而越容易产生内存溢出异常。其实原因很简单,因为操作系统分配给每个进程的内存是有限制的,比如32位的windows限制为2G(实际上我们在使用的时候设置-Xmx并不能设置为2G,一般只有1.5G),减去Xmx(最大堆容量),在减去MaxPermSize(最大方法区容量),程序计数器消耗内存很小,忽略不计,如果虚拟机本身耗费的内存不计算在内,那么剩下的就由虚拟机栈和本地方法栈瓜分了。因此如果是由于建立过多线程导致的内存溢出,那么可以通过减少最大堆和减少栈容量来获取更多的线程。

### 方法区和运行时常量池溢出

#### 运行时常量池导致的内存溢出异常
如果在JDK6的环境下写下如下的代码,那么会抛出OutOfMemoryError: PermGen space这样的错误信息。
```java
//VM Args：-XX:PermSize=10M -XX:MaxPermSize=10M
public static void main(String[] args) {
    // 使用List保持着常量池引用,避免Full GC回收常量池行为
    List<String> list = new ArrayList<String>();
    // 10MB的PermSize在integer范围内足够产生OOM了
    int i = 0; 
    while (true) {
        list.add(String.valueOf(i++).intern());
    }
}

```
而使用JDK7就不会出现这样的问题,因为在JDK6的intern()方法会把首次遇到的字符串实例复制到永久带中,返回的也是永久带中这个字符串实例的引用。而JDK7的intern()实现不会再复制实例,只是在常量池中记录首次出现的实例引用。

#### 方法区出现内存溢出异常
我们知道方法区用于存放Class的相关信息,如类名、访问修饰符、常量池、字段描述、方法描述等。如果在运行的时候大量的类填满方法区,那么也会抛出OutOfMemoryError,比如下面的这段代码就借助了CGLib(2.0版本)使得方法区出现内存溢出异常。
```java
@Test
  //VM Args：-XX:PermSize=10M -XX:MaxPermSize=10M
  public void permGenOutOfMemory() {
    //如果没有这个设置在JDK7中无法打印出异常信息
    Thread.setDefaultUncaughtExceptionHandler(new MyHandler());
    while (true) {
      Enhancer enhancer = new Enhancer();
      enhancer.setSuperclass(OOMObject.class);
      enhancer.setUseCache(false);
      enhancer.setCallback(new MethodInterceptor() {
       @Override
       public Object intercept(Object obj, Method method, Object[] args, MethodProxy proxy)
           throws Throwable {
         return proxy.invokeSuper(obj, args);
       }
      });
      enhancer.create();
    }
}
class MyHandler implements UncaughtExceptionHandler {

  @Override
  public void uncaughtException(Thread t, Throwable e) {
    System.out.println("异常信息为:" + e);
  }
}
public class OOMObject {
  public OOMObject() {
  }
}
```
抛出的异常信息为：
```
异常信息为:java.lang.OutOfMemoryError: PermGen space
```
当然我们在平时做WEB项目的时候,如果在启动项目的时候指定-XX:MaxPermSize和-XX:PermSize的值都比较小的情况下,ClassLoader在装载较多类的时候也是会抛出这个异常的,有兴趣的读者可以试一试。

### 本机直接内存溢出
DirectMemory可以通过-XX:MaxDirectMemorySize指定,如果不指定则默认与Java堆最大值(-Xmx指定)一样。
```java
//-Xmx10M -XX:MaxDirectMemorySize=10M
public static void main(String[] args) throws IllegalArgumentException, IllegalAccessException {
    Field unsafeField = Unsafe.class.getDeclaredFields()[0];
    unsafeField.setAccessible(true);
    Unsafe unsafe = (Unsafe) unsafeField.get(null);
    int _1MB = 1024*1024;
    while(true) {
      unsafe.allocateMemory(_1MB);//申请分配内存的方法
    }
```

抛出的异常信息为:
```
Exception in thread "main" java.lang.OutOfMemoryError
    at sun.misc.Unsafe.allocateMemory(Native Method)
    at test.OutofMemoryTest.main(OutofMemoryTest.java:65)
```
由DirectMemory导致的内存溢出,一个明显的特征是在Heap Dump文件中不会看见明显的异常,如果发现OOM之后Dump文件很小,而程序中又直接或间接使用了NIO,那就可以考虑检查一下是不是这方面的原因。

### 关于OutOfMemory的排查
大多数时候当我们遇到了OutOfMemoryError的时候,一般可以通过提示信息确定到是哪里的问题,一般都可以通过调整堆、方法区的大小来保证程序的正常运行。但是有时候内存泄露导致的问题就不是简单的通过调整堆大小可以解决的了。不过JAVA自带了许多关于排查问题的工具,特别是线上问题,通过这些命令都很有帮助。比如说jmap、jstat、jinfo、jps、jstack这样的命令工具,如果觉得不方便还可以使用jvisualvm、jconsole这样的图形化工具,当然我们常用的Eclipse也提供了Memory Analyzer这样的分析工具。

### 写在最后
本文中代码测试环境为JDK1.7(64位),Windows64位,Eclipse Luna Service Release 2 (4.4.2)
### 参考资料
《深入理解Java虚拟机第二版》