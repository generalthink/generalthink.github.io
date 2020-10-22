---
title: Java对象在内存中的布局
date: 2020-09-25 15:53:04
tags: jvm
---

### 写在前面

Java是用C++写的，所以java对象最终会映射到c++中的某个对象，用这个对象可以描述所有Java对象。而我们所熟知的synchronized锁的优化就是基于这个对象来实现的。


### 对象在内存中的布局

Java对象在被创建的时候,在内存分配完成后，虚拟机需要对对象进行必要设置， 例如这个对象是哪个类的实例、如何才能找到类的元数据信息、对象的哈希码、对象的GC分代年龄等信息。

这些信息存放在对象的对象头(Object Header)中。根据虚拟机当前运行状态的不同，如是否启用偏向锁等对象头会有不同的设置方式。


**在虚拟机中，对象在内存中存储的布局可以分为3块区域：对象头(Header)、实例数据(Instance Data)和对齐填充(Padding)**

<!--more-->

#### 对象头

HotSpot虚拟机对象头包括两部分信息，第一部分用于存储对象自身的运行时数据，如哈希码、GC分代年龄、锁状态标志、线程持有的锁、偏向线程ID、偏向时间戳。

这部分数据的长度在32位和64位的虚拟机中分别是32bit和64bit,官方称为"Mark Word"。 考虑到虚拟机的空间效率，Mark Word被设计成一个非固定的数据结构以便在极小的空间内存储尽量多的信息。

对象头另一部分是类型指针，即指向对象的类元数据，虚拟机通过这个指针确定该对象是哪个类的实例。

我们可以在JVM源码(hotspot/share/oops/markOop.hpp)中看到对象头中存储内容的定义

```c
class markOopDesc: public oopDesc {
 public:
  enum { age_bits             = 4,
         lock_bits            = 2,
         biased_lock_bits     = 1,
         max_hash_bits        = BitsPerWord - age_bits - lock_bits - biased_lock_bits,
         hash_bits            = max_hash_bits > 31 ? 31 : max_hash_bits,
         cms_bits             = LP64_ONLY(1) NOT_LP64(0),
         epoch_bits           = 2
  };
}
```

![对象头](/images/macos-jvm/markOop-object-header.png)

+ hash: 对象的哈希码
+ age: 对象的分代年龄
+ biased_lock : 偏向锁标识位
+ lock: 锁状态标识位
+ JavaThread* : 持有偏向锁的线程ID
+ epoch: 偏向时间戳

例如在32位的HotSpot虚拟机中，如果对象处于未被锁定的状态下，那么Mark Word的32bit空间的25bit用于存储对象哈希码，4bit用于存储对象分代年龄，2bit用于存储锁标志位，1bit固定为0
而在其他状态(轻量级锁，重量级锁，GC标记，可偏向)下对象的存储内容如下表所示

![32位系统Mark Word](/images/macos-jvm/mark-word-32-system.png)

#### 实例数据
实例数据部分是对象真正存储的有效信息，也是在程序代码中所定义的各种类型的字段内容。

#### 对其填充
第三部分对其填充并不是必然存在的，也没有特别的含义，仅是占位符的作用，因为HotSpot VM的内存管理系统要求对象起始地址必须是8字节的整数倍，换句话说，就是对象的大小必须是8字节的整数倍。而对象头部分正好是8字节的倍数，因此当对象实例数据部分没有对齐时，就需要通过对齐填充来补全。


### 查看对象头

我们可以通过openjdk的jol工具来查看对象头存储的内容，首先代码如下

```xml
<dependency>
  <groupId>org.openjdk.jol</groupId>
  <artifactId>jol-core</artifactId>
  <version>0.9</version>
</dependency>

```


```java
public class GetRange {
  // -XX:-UseCompressedOops
  public static void main(String[] args) {

    // 1字节=8位(1byte = 8bit)
    System.out.println(VM.current().details());

    MyClass myClass = new MyClass();

    ClassLayout classLayout = ClassLayout.parseInstance(myClass);

    System.out.println("****New Object****");

    System.out.println(classLayout.toPrintable());

    int hashCode = myClass.hashCode();

    System.out.println("MyClass hashCode : " + hashCode) ;
    System.out.println("MyClass hashCode 二进制 " + Integer.toBinaryString(hashCode));
    System.out.println("MyClass hashCode 二进制长度 " + Integer.toBinaryString(hashCode).length());

    System.out.println();

    System.out.println("****After invoke hashCode()****");

    System.out.println(classLayout.toPrintable(myClass));

    // 获取系统字节序
    System.out.println("系统当前字节序是：" + ByteOrder.nativeOrder());

  }
}

class MyClass {

  String name = "think123";

  int[] other;

  boolean status;
}

```

输出内容如下

![对象头](/images/macos-jvm/jol-object-header.png)

可以知道对象头占了12个字节,存在3个字节的对其填充。

jvm中使用oopDesc来描述一个对象

```c
class oopDesc {
 private:
  volatile markOop _mark;
  union _metadata {
    Klass*      _klass;
    narrowKlass _compressed_klass;
  } _metadata;

}
```

我们可以看到对象头有两部分，mark 部分，官方称为 mark word,存储的是哈希码，对象分代年龄，偏向锁标志等信息。mark word长度是一个系统字宽，在64bit系统上是8个字节。

第二部分是klass的类型指针，指向这个对象是哪个类的实例,这里使用的是union这个联合体，表示变量`_klass`和`_compressed_klass`共享同一段内存。未开启指针压缩时使用`_klass`，开启指针压缩时使用`_compressed_klass`。narrowKlass实际上是一个32bit的unsigned int类型，因此占用4个字节，所以开启指针压缩后对象的头部长度整体为12字节。

> narrowKlass定义在hotspot/src/share/vm/oops/oopsHierarchy.hpp,它是juint类型，实际上是32bit的unsigned int

对象体：MyClass中定义了三个字段,int[],String都占用4个字节长度,boolean类型的占用了1个字节长度

填充部分：上面对象头和对象体长度和为21字节，因为要8字节对齐，因此需要填充3字节，这样就刚好等于24字节。

当我们关闭指针压缩后(-XX:-UseCompressedOops),mark word占16个字节，同时也采用8字节进行对其。name和other两个数据域分别占据8字节。而开启指针压缩之后，这两个字节分别占用4个字节。
所以开启这个选项是可以节约内存的，从jdk8之后已经默认开启此选项。

![未开启指针压缩](/images/macos-jvm/markword-no-compress.png)


上面我把当前系统的字节序打印了出来，可以看到当前是小字节序。

举例来说，数值0x2211使用两个字节储存：高位字节是0x22，低位字节是0x11。
大端字节序：高位字节在前，低位字节在后，这是人类读写数值的方法。
小端字节序：低位字节在前，高位字节在后，即以0x1122形式储存。

因此我们查看myClass的hashCode的时候就要倒着看了。我们查看object header中的hashcode要从第9位到39位(64位系统中,用31字节保存哈希)开始查看。


```

myClass的hashcode(长度为30,最高位补2个0) :

myClass HashCode :    00100001 01011000 10001000 00001001

Mark Word中HashCode : 00001001 10001000 01011000 0 0100001

```

可以发现myClass的hashCode和mark word中的存储刚好是反着来的。

不是说存储hash的只有31位吗？为什么这里用32位来比较呢？我用32位来比较是为了更加易于观察。 实际上MarkWord中保存哈希码最后8位的第一位0是从未使用的25位中借来的(需要结合小字节序)


接下来，我们使用JOL工具查看处于不同锁状态下，mark word中的标志位是怎样的。

### 偏向锁

首先我们来看偏向锁，由于JVM默认会在启动后4秒才会启动偏向锁，所以测试时需要设置马上启动偏向锁(-XX:BiasedLockingStartupDelay=0)

```java
// -XX:BiasedLockingStartupDelay=0
public static void main(String[] args) throws InterruptedException {

    Layouter layouter = new HotSpotLayouter(new X86_32_DataModel());

    MyClass myClass = new MyClass();

    ClassLayout layout = ClassLayout.parseInstance(myClass);

    System.out.println("进入同步代码块之前:");
    System.out.println(layout.toPrintable());


    synchronized (myClass) {
      System.out.println("同步代码块中:");
      System.out.println(layout.toPrintable());
    }

    System.out.println("退出同步代码块后:");
    System.out.println(layout.toPrintable());

}

```
![偏向锁](/images/macos-jvm/biased-lock.png)

1. 可以看到在进入同步代码块之前低8位是00000101,表示处于偏向锁(后三位是101),但是现在还没有偏向任何一个线程，因此没有数据
2. 处于同步代码块中以及退出同步代码块时,mark word一致，低8位都是00000101,处于偏向锁，且存储了偏向线程地址以及时间戳等信息。说明退出同步块后，依然保留偏向锁的信息

### 轻量级锁以及重量级锁

我们使用下面的代码来演示轻量级锁以及重量级时状态位的变化情况

```java
 //不设置立刻启动偏向锁
public static void main(String[] args) throws InterruptedException {

    Layouter layouter = new HotSpotLayouter(new X86_32_DataModel());

    MyClass myClass = new MyClass();

    ClassLayout layout = ClassLayout.parseInstance(myClass);

    System.out.println("创建t1线程之前:");
    System.out.println(layout.toPrintable());

    Thread t1 = new Thread(() -> {
       synchronized ((myClass)) {
         try {
             TimeUnit.SECONDS.sleep(5);
         } catch (InterruptedException e) {
             e.printStackTrace();
         }
       }
    });

    t1.start();

    System.out.println("持有锁之前:");
    System.out.println(layout.toPrintable());


    synchronized (myClass) {
        System.out.println("持有锁中:");
        System.out.println(layout.toPrintable());
    }

    System.out.println("释放锁:");
    System.out.println(layout.toPrintable());

    System.out.println("System.gc() 后");
    System.gc();
    System.out.println(layout.toPrintable());

}

```

运行结果如下：

![](/images/macos-jvm/lock-change-in-mark-word-1.png)![轻量级锁和重量级锁](/images/macos-jvm/lock-change-in-mark-word-2.png)


mark word的状态变更过程如下:

1. 创建t1线程前阶段，锁标志位为01，偏向锁标志位为0，处于无锁状态
2. main线程持有锁之前阶段，标志位为00，属于轻量级锁，**此时t1线程已经持有锁且main线程未请求锁，所以此时无竞争**
3. main线程持有锁阶段，标志位为10，膨胀为重量级锁，**此时t线程已经已经释放了锁，main线程获取锁成功，即此时存在竞争**
4. main线程释放锁阶段，标志位为10，仍为重量级锁，**不会自动降级**
5. **System.gc后阶段，恢复为无锁状态，GC年龄变为1**

至此，我们知道了处于不同锁情况下，状态位的变化情况。