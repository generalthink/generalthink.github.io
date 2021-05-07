---
title: Java中对象在JVM中如何表示？
date: 2021-04-27 17:20:37
tags: [java, jvm]
---



我们写代码的时候，前端传递参数给后端，后端都会有一个对象来负责参数接收，同样的JVM内部也有一个模型来表示Java对象，而这个就是oop-Klass模型。

Hotspot虚拟机在内部使用两组类来表示Java的类和对象

1. oop(ordinary object pointer)用来描述对象实例信息
2. kclass用来描述Java类，是虚拟机内部Java类型结构的对等体

![class-in-jvm](/images/java/class-in-jvm.png)

<!--more-->

JVM内部基于OOP-Klass模型描述一个Java类，将一个Java类一拆为二，第一个是oop,第二个是klass.

oop是ordinary object pointer(普通对象指针), 它用来表示对象的实例信息(Java类实例对象中各个属性在运行期的值)。看起来像是一个指针，而实际上对象实例数据都藏在指针所指向的内存首地址后面的一篇内存区域中。


![oopDesc](/images/java/oopDesc.png)

> 以上代码在https://github.com/openjdk/jdk/blob/jdk8-b29/hotspot/src/share/vm/oops/oop.hpp

从注释中我们可以知道oopDesc是对象类的顶级基础类，{name}Desc用来描述Java对象格式，从而可以在C++中访问到它。
Java中万物皆为对象，所以C++中将方法，常量，数组等都抽象成为了对象，可以用oop来表示


![](/images/java/other-oops.png)

而klass则包含元数据和方法信息，用来描述Java类或者JVM内部自带的C++类型信息。比如Java类的继承信息、成员变量、静态变量、成员方法、构造函数等信息都在klass中保存，JVM根据这个
可以在运行期反射出Java类的全部结构信息。JVM根据这个就能在运行期反射出Java类的全部结构信息。


同样的和oop一样，klass也是成体系的，它也有很多子类，这里我就不列举出来了，我们可以看一个instanceKlass的定义


![layout](/images/java/instanceKlass-layout.png)

由于mirror也是一个instanceKlass，所以它包含了instanceKlass所包含的一切字段

执行 `new A()` 的时候，JVM native 层里发生了什么。首先，如果这个类没有被加载过，JVM 就会进行类的加载，并在 JVM 内部创建一个 instanceKlass 对象表示这个类的运行时元数据（相当于 Java 层的 Class 对象）。到初始化的时候（执行 invokespecial A::<init>），JVM 就会创建一个 instanceOopDesc 对象表示这个对象的实例，然后进行 Mark Word 的填充，将元数据指针指向 Klass 对象，并填充实例变量。

根据对 JVM 的理解，我们可以想到，元数据—— instanceKlass 对象会存在元空间（方法区），而对象实例—— instanceOopDesc 会存在 Java 堆。Java 虚拟机栈中会存有这个对象实例的引用。

### handle体系

JVM内部访问对象并不是直接通过oop，而是通过handle,handle封装了oop,handle是对普通对象的一种间接引用，那JVM内部为什么要什么这种间接引用呢？

这完全是为GC考虑。

1. 通过handle，能够让GC知道其内部代码有哪些地方持有GC管理对象的引用，只需要臊面handle对应的table,这样JVM无须关注其内部哪些地方持有对普通对象的引用。
2. GC过程中，如果发生了对象移动(比如从新生代移动到老年代),那么JVM内部引用无须跟着更改为被移动对象的新地址，JVM只需要更改handle table里面对应的指针即可。

在JVM中为了方便回收oop和klass(oop在堆中,klass处于metaspace),会将这两个对象封装成 oop


![](/images/java/handle.png)

> https://github.com/openjdk/jdk/blob/jdk8-b29/hotspot/src/share/vm/runtime/handles.hpp

在handles.hpp被编译后，会分别出现oop和klass对应的handle,

比如 

```
# 类实例handle
instanceHandle

# 方法实例handle
methodHandle

......

# 类元结构handle
instanceKlassHandle

# 方法元结构handle
methodKlassHandle

......

```

jvm在具体描述一个类型时，会使用oop(其实这里是oopDesc)去存储这个类型的实例数据，使用klass去存储这个类型的元数据和虚方法表。当一个类型完成其生命周期后，需要将这两部分都回收
，因此将 oop封装成oop,klass也封装成oop。

可能有点绕，换个说法。

比如我们将JVM内部描述类信息模型叫做data-meta模型，将jvm内部的oop体系的类名全部改成Data结尾，比如instanceData, methodData。 klass体系的类改成以Meta结尾，比如 methodMeta, instanceMeta, JVM再进行GC时，即要回收Data类实例，也要回收Meta类实例，为了让GC方便回收，因此对于每一个Meta和Data类，JVM在内部将其封装成了oop模型。

对于Data类，内存布局前面是oop对象头，后面紧跟实例数据；对于Meta类，其内存布局是前面是oop对象头，后面紧跟实例数据和方发表。 封装成oop之后，再进一步使用handle来封装，于是有利于GC内存回收。




### 查看Java对象在内存中的样子

为了让梦想照进现实，我们使用HSDB来查看JAVA对象在JVM中长什么样子,我们使用下面的代码来进行测试

```java
public class MyClassTest {

  private static String name = "think123";
  private static int age = 18;
  private static String staticString = "test";
  private String desc = "heheda";

  public static void main(String[] args) 
                      throws IOException {
    MyClassTest test = new MyClassTest();
    hello("zhangsan");
    System.in.read();
  }
  private static String hello(String otherName) {
    return name + " : " + otherName;
  }
}
```

我的java是安装在 C:\Program Files\Java\jdk1.8.0_172\lib，所以在这个目录下执行以下命令启动HSDB

```
java -cp sa-jdi.jar sun.jvm.hotspot.HSDB

```

然后启动我们的程序(启动时最好关闭指针压缩：-XX:-UseCompressedOops)， 使用`jps -l `命令查看java进程id, 然后选择 `File --> Attach to Hotspot Process`,输入java进程id

![attach](/images/java/hsdb-attach.gif)
![](/images/java/hsdb-threads.png)

#### 查看OOP

此时我们可以查看main线程的oop以及线程堆栈，在堆栈中可以看到我们的MyClassTest对象分配到了年轻代，也可以看到对应的内存地址，然后把这个地址复制下来，我们可以通过

Tools--> Inspector查看对应的oop对象。

![](/images/java/show-thread-info.gif)


不过这种方式比较麻烦，我们可以使用`Tools --> Object Histogram`的方式来查看
![查找对象](/images/java/hsdb-object.png)

找到对应的对象后，双击打开，然后点击Inspect

![](/images/java/hsdb-object-type.png)


![](/images/java/hsdb-oop.png)

#### 查看Klass

![](/images/java/hsdb-class-browser.png)

![](/images/java/hsdb-klass.png)

而在Method中我们则可以看到方法对应的字节码
![](/images/java/hsdb-method-bytecode.png)

常量池中这是MyClass中的常量(比如引用类全限定名，方法名，常量，LocalNumberTable,LocalVariableTable等)


通过刚才查看到的地址，我们还可以看看Klass的数据结构是怎样的

![](/images/java/hsdb-klass-structure.png)
> klass中的Method的定义: https://github.com/openjdk/jdk/blob/61bb6eca3e34b3f8382614edccd167f7ecefba65/src/hotspot/share/oops/method.hpp

可以看到，它的结构和我们最开始列出来的instanceKlass layout是一样的。


### 总结

我们需要记住的是在JVM内部实例对象和Class是分别使用不同的数据结构来表示的(oop,Klass),我们可以借助HSDB工具来查看它们在JVM中的数据结构。

而我们常用到的反射，只所以能拿到那些字段，方法，接口等这些参数全是因为借助了Klass来实现的。

同时我们还可以借助它来查看常量池，比如困扰大家的intern问题，就可以结合HSDB来分析。


### think123

以前最开始看小说的时候，那会儿异界修仙流比较出名，过程倒是记不住了，但是各个等级还是印象颇深。

炼气  筑基  金丹  元婴  离合  渡劫  大乘

如果程序员也有这种级别认定的话，那或许也是一件不错的事儿,每次升级都大吼一声

"键来!"


### 参考书籍

《揭秘Java虚拟机-JVM设计原理与实现》