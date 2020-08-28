---
title: 深入浅出String.intern()
date: 2020-08-26 16:48:35
tags: java
---

### 常量池

Java虚拟机管理的内存包含以下几个运行时数据区域

![Java虚拟机运行时数据区](/images/thread-java-running-data-area.png)

方法区与java堆一样，是各个线程共享的内存区域，用于存储已经被虚拟机加载的类型信息、常量、静态变量、即时编译器编译后的代码缓存等数据。

**JDK8之前很多人叫它永久代(这里可以联想下年前代，老年代)，是因为当时HotSpot虚拟机的设计团队选择将收集器的分代设计扩展至方法区，或者说使用永久代来实现方法区而已。
这样使得HotSpot的垃圾收集器能像管理Java堆一样管理这部分内存。但是对于其他虚拟机实现是不存在这个概念的。**


运行时常量池(Runtime ConstantPool)是方法区的一部分。Class文件中除了有类的版本、字、方法、接口等描述信息外,还有一项信息是常量池表(ConstantPoolTable),用于存放编译期生
成的各种字面量与符号引用,这部分内容将在类加载后存放到方法区的运行时常量池中。

> String str = "think123" 其中的think123就直接保存在常量池中

<!--more-->

运行时常量池存了在编译器能产生之外，在运行期间也可以将新的常量放入池中,这种特性被利用最多的就是String类的intern()方法


`String::intern()`是一个本地方法,它的作用是如果字符串常量池中已经包含一个等于此String对象的字符串,则返回代表池中这个字符串的String对象的引用;否则,会将此String对象包含的字符串添加到常量池中,并且返回此String对象的引用。

伪代码类似如下:

```

if(StringPool.contains(search_string)) {
  
  return search_string_addr;
}

search_string_addr = StringPool.add(search_string);

return search_string_addr;

```

### 神奇的代码

下面的代码你猜猜会输出什么？


```java
public class Main {

  public static void main(String[] args) {

    String s1 = new StringBuilder().append("think").append("123").toString();

    System.out.println(s1.intern() == s1);

    String s2 = new StringBuilder().append("ja").append("va").toString();

    System.out.println(s2.intern() == s2);
  }
}
```

1. JDK6 : false   false

2. JDK8 : true    false

3. JDK12: true    true


神不神奇? 接下来就是见证奇迹的时刻了。


### 为什么会有不同的结果

从JDK6开始就就有放弃永久代，逐步改为采用本地内存来实现方法区的计划了。

到了JDK7的HotSpot，已经把原本放在永久代的字符串常量池、静态常量池等移出到堆中了

而到了JDK8,完全废弃了永久代的概念，改用在本地内存中实现的元空间(meta space)来代替了。把JDK7中永久代还剩下的内容(主要是类型信息)全部移到元空间中。

#### JDK6为什么会这样？

在JDK6中，常量池还是存在于永久代中。

也就是说, **常量池和堆是两块不同的内存区域。**


在JDK6中，intern()方法会把首次遇到的字符串实例复制到永久代的字符串常量池中，返回的也是永久代里面这个字符串实例的引用，而StringBuilder创建的字符串对象实例是在Java堆上，这必然不可能是同一个引用，所以返回false

#### JDK7+为什么会有这样的结果

我本地没有JDK7环境,但是JDK7和JDK8输出是一样的结果。那么为什么JDK7会输出true false呢？

因为从JDK7开始，字符串常量池已经被移到了堆当中了。


对于变量s1,虽然常量池中不存在"think123",但是堆中是存在的。 当调用s1.intern()的时候，实际是将堆中think123放入了常量池中。所以s1.intern()和s1都是指向java堆上的String对象

对于变量s2,常量池中一开始就存在"java"字符串，s2.intern()方法返回的是常量池中的字符串对象，而s2是new的字符串对象，虽然都在堆中，但是并不是一个对象。所以输出false

> 这个一开始就存在常量池的java字符串在sun.misc.Version类中，具体可见R大分析 https://www.zhihu.com/question/51102308/answer/124441115
> 周志明老师在第三版《深入理解java虚拟机》已经把这个坑填上了

#### JDK12为什么会是这样的结果？

难道从JDK12开始常量池又不在堆里面了？ 当然不是这是因为sun.misc.Version这个类在JDK12(JDK11也不存在)中已经不存在了。这里我debug了源码，将部分JDK12中存在的字符串常量打印出来了(全部有1000多个)

```
wh.resolve()->print_value_string() = (char *) ""Cannot suppress a null exception."{0x0000000787f00fb0}"
wh.resolve()->print_value_string() = (char *) ""Self-suppression not permitted"{0x0000000787f01000}"
wh.resolve()->print_value_string() = (char *) ""Caused by: "{0x0000000787f01048}"
wh.resolve()->print_value_string() = (char *) ""Suppressed: "{0x0000000787f01080}"
wh.resolve()->print_value_string() = (char *) ""<init>"{0x0000000787f02de8}"
wh.resolve()->print_value_string() = (char *) ""sun.net.www.protocol"{0x0000000787f03a40}"
wh.resolve()->print_value_string() = (char *) ""java.protocol.handler.pkgs"{0x0000000787f03a80}"
wh.resolve()->print_value_string() = (char *) ""null"{0x0000000787f04828}"
wh.resolve()->print_value_string() = (char *) ""-2147483648"{0x0000000787f04858}"
wh.resolve()->print_value_string() = (char *) ""system"{0x0000000787f06730}"
wh.resolve()->print_value_string() = (char *) ""*"{0x0000000787f073e0}"
wh.resolve()->print_value_string() = (char *) ""classLoader"{0x0000000787f074a0}"
wh.resolve()->print_value_string() = (char *) ""security"{0x0000000787f074f0}"
wh.resolve()->print_value_string() = (char *) ""float"{0x0000000787f07a38}"
wh.resolve()->print_value_string() = (char *) ""double"{0x0000000787f07aa0}"
wh.resolve()->print_value_string() = (char *) ""segments"{0x0000000787f08210}"
wh.resolve()->print_value_string() = (char *) ""segmentMask"{0x0000000787f08518}"
wh.resolve()->print_value_string() = (char *) ""int"{0x0000000787f085a0}"
wh.resolve()->print_value_string() = (char *) ""segmentShift"{0x0000000787f08778}"
wh.resolve()->print_value_string() = (char *) ""sizeCtl"{0x0000000787f087c0}"
wh.resolve()->print_value_string() = (char *) ""transferIndex"{0x0000000787f087f0}"
wh.resolve()->print_value_string() = (char *) ""baseCount"{0x0000000787f08828}"
wh.resolve()->print_value_string() = (char *) ""cellsBusy"{0x0000000787f08860}"
wh.resolve()->print_value_string() = (char *) ""value"{0x0000000787f08910}"
wh.resolve()->print_value_string() = (char *) ""jdk.internal.ref.Cleaner"{0x0000000787f08e50}"
wh.resolve()->print_value_string() = (char *) ""Reference Handler"{0x0000000787f09228}"
wh.resolve()->print_value_string() = (char *) ""Finalizer"{0x0000000787f097d8}"
wh.resolve()->print_value_string() = (char *) ""java.home"{0x0000000787f0b1d0}"
wh.resolve()->print_value_string() = (char *) ""Finalizer"{0x0000000787f097d8}"
wh.resolve()->print_value_string() = (char *) ""java.home"{0x0000000787f0b1d0}"
wh.resolve()->print_value_string() = (char *) ""user.home"{0x0000000787f0b208}"
wh.resolve()->print_value_string() = (char *) ""user.dir"{0x0000000787f0b260}"
wh.resolve()->print_value_string() = (char *) ""user.name"{0x0000000787f0b2b0}"
wh.resolve()->print_value_string() = (char *) ""sun.jnu.encoding"{0x0000000787f0b308}"
wh.resolve()->print_value_string() = (char *) ""file.encoding"{0x0000000787f0b360}"
wh.resolve()->print_value_string() = (char *) ""os.name"{0x0000000787f0b3b8}"
wh.resolve()->print_value_string() = (char *) ""os.arch"{0x0000000787f0b408}"
wh.resolve()->print_value_string() = (char *) ""os.version"{0x0000000787f0b458}"
wh.resolve()->print_value_string() = (char *) ""line.separator"{0x0000000787f0b4b0}"
wh.resolve()->print_value_string() = (char *) ""file.separator"{0x0000000787f0b508}"
wh.resolve()->print_value_string() = (char *) ""path.separator"{0x0000000787f0b560}"
wh.resolve()->print_value_string() = (char *) ""java.io.tmpdir"{0x0000000787f0b5b8}"
wh.resolve()->print_value_string() = (char *) ""http.proxyHost"{0x0000000787f0b610}"
wh.resolve()->print_value_string() = (char *) ""http.proxyPort"{0x0000000787f0b648}"
wh.resolve()->print_value_string() = (char *) ""https.proxyHost"{0x0000000787f0b680}"
wh.resolve()->print_value_string() = (char *) ""https.proxyPort"{0x0000000787f0b6b8}"
wh.resolve()->print_value_string() = (char *) ""ftp.proxyHost"{0x0000000787f0b6f0}"
wh.resolve()->print_value_string() = (char *) ""ftp.proxyPort"{0x0000000787f0b728}"
wh.resolve()->print_value_string() = (char *) ""id"{0x0000000787953030}"
wh.resolve()->print_value_string() = (char *) ""language"{0x0000000787953218}"
wh.resolve()->print_value_string() = (char *) ""country"{0x0000000787953270}"
wh.resolve()->print_value_string() = (char *) ""variant"{0x00000007879532c8}"
wh.resolve()->print_value_string() = (char *) ""hashcode"{0x0000000787953320}"
wh.resolve()->print_value_string() = (char *) ""script"{0x0000000787953378}"
wh.resolve()->print_value_string() = (char *) ""extensions"{0x00000007879533d0}"
wh.resolve()->print_value_string() = (char *) ""Main"{0x0000000787957368}"
wh.resolve()->print_value_string() = (char *) ""main"{0x0000000787957570}"

... 省略 ....

// think 123是程序代码中出现的常量,实际上并非原生
wh.resolve()->print_value_string() = (char *) ""think"{0x0000000787957a90}"
wh.resolve()->print_value_string() = (char *) ""123"{0x0000000787957ac0}"
wh.resolve()->print_value_string() = (char *) ""think123"{0x0000000787957af0}"
```

上面是我使用CLION 调试JVM源码输出的内容，其中的字符串就是常量表中的内容。在jdk12中如果你将java换成上面任意出现的一个字符串输出结果都会是false(JDK8中大部分都可以)


### intern源码分析

以下代码是基于openJdk 12,和JDK8有细微区别，但是总体逻辑是一致的。

先搜索java_lang_string_intern字符串找到intern方法的实现入口String.c文件中

```c++
JNIEXPORT jobject JNICALL
Java_java_lang_String_intern(JNIEnv *env, jobject this)
{
    return JVM_InternString(env, this);
}

```

最终会调用StringTable的intern方法

```c++

oop StringTable::intern(Handle string_or_null_h, const jchar* name, int len, TRAPS) {
  
  // 首先计算hashcode
  unsigned int hash = java_lang_String::hash_code(name, len);

  // 通过hashcode以及name(实际是将string转换成了unicode)在StringTable中的share_table寻找字符串
  oop found_string = StringTable::the_table()->lookup_shared(name, len, hash);
  if (found_string != NULL) {
    return found_string;
  }
  if (StringTable::_alt_hash) {
    hash = hash_string(name, len, true);
  }

  // 未找到就加入StringTable
  return StringTable::the_table()->do_intern(string_or_null_h, name, len,
                                             hash, CHECK_NULL);
}
```

在找寻字符串的时候，lookup_shared()方法实际上就是在HashTable中去寻找这个字符串,通过hash确定StringTable中桶的位置，通过name作为key来确定value.

实际上无论字符串是否在常量池,shared_table的大小都是为0，也就是说这个fonund_string一直为NULL


如果字符串未找到,那么就会调用do_intern()方法将其加入StringTable中。(JDK8和JDK12有些许区别，但是原理是一样的)

![do_intern](/images/java/stringtable-dointern.png)

这里的string_or_null_h是一个Handle对象，为什么访问对象的时候要使用它呢？是因为垃圾回收时对象可能被移动(对象地址发生改变)，通过handle访问对象可以对使用者屏蔽垃圾回收细节。


你需要注意的是红框中的while循环,如果我们的字符串在常量池中存在，那么会

![insert_or_get](/images/java/insert_or_get_string.png)


需要注意的是这里的_local_table->get和上面的lookup_shared不同之处在于lookeup_shared中的map是一个强类型的map(里面的数据不会被垃圾回收),而_local_table中的则是是weak的，在内存不足的时候是会被回收的。


### 常量池是如何被加入到常量池的？

在启动JVM的时候就会解析ldc这个指令(可以通过javap查看对应解析常量池的指令)，最终还是会通过intern()方法将其加入到常量池中,更具体的调用我就不做过多介绍了，这里我把调用堆栈贴上来

![解析ldc指令](/images/java/resolve_ldc.png)


### 总结

实际开发中其实不需要去记住常量池中已经加载了哪些字符串常量，你只需要记住

1. 通过双引号声明的字符串，直接保存在常量池中

2. new的字符串在堆中,JDK6中常量池在永久代,JDK7+将常量池放到了堆中

3. 常量池相当于一个缓存,intern()方法实际上就是动态将字符串常量加入到这个缓存中。类似HashMap::computeIfAbsent方法

4. 一味的使用intern并不好，比较常量池的实现是hashtable,它也是有容量限制的，如果往里面放入大量的内容，会发生大量的hash冲突,导致运行速度过慢


