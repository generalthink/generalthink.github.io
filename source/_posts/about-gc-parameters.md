title: 关于GC参数的一个问题
date: 2016-11-30 21:23:32
tags: java
categories: [java,jvm]
description:
keywords: java,jvm,gc参数设置
---
### 我遇到一个一个问题
我遇到了一个问题,这个问题真的很难,我想说在国内去关注这个问题的人应该并不多，至少我问了很多人但是他们并没有去深究这个问题，不过既然让我发现了这个问题那么我肯定不能带着疑惑离开，我必须得留下点什么，所以我解决了这个问题，然后写下了这篇博客，我都不知道应该怎么赞美自己了,好了不吹牛逼了。。。

这个问题是这样的，我们都知道可以设置VM参数来设置堆的大小,通过-Xmx设置堆的最大值,-Xms设置堆的最小值。然后虚拟机在启动的时候会根据我们的参数进行动态调整。但是这其中还是有点小秘密的。

我们在本地运行这样的一段代码:
```java
//VM Args：-Xms4m  -Xmx4m -XX:+PrintGCDetails
  @Test
  public void heapOutOfMomory() {
    Byte[] b = new Byte[1*1024*1024];
  }
```
查看打印出来的的GC信息：
```
[GC [PSYoungGen: 2344K->312K(2368K)] 2910K->1477K(5120K), 0.0083535 secs] [Times: user=0.05 sys=0.00, real=0.01 secs] 
Heap
 PSYoungGen      total 2368K, used 1830K [0x00000000ffd60000, 0x0000000100000000, 0x0000000100000000)
  eden space 2048K, 74% used [0x00000000ffd60000,0x00000000ffedb9d8,0x00000000fff60000)
  from space 320K, 97% used [0x00000000fffb0000,0x00000000ffffe010,0x0000000100000000)
  to   space 320K, 0% used [0x00000000fff60000,0x00000000fff60000,0x00000000fffb0000)
 ParOldGen       total 5504K, used 5261K [0x00000000ff800000, 0x00000000ffd60000, 0x00000000ffd60000)
  object space 5504K, 95% used [0x00000000ff800000,0x00000000ffd23760,0x00000000ffd60000)
 PSPermGen       total 21248K, used 5716K [0x00000000fa600000, 0x00000000fbac0000, 0x00000000ff800000)
  object space 21248K, 26% used [0x00000000fa600000,0x00000000fab95308,0x00000000fbac0000)
```

发现了吧！堆的大小是5504(ParOldGen)+320(from)+320(to)+2048(eden) = 8192(8M),虽然我们设置的是4M,但是实际上却是8M?那么这个为什么了?我们的参数没有起作用?

### 和操作系统有关的VM
我们先看一个图,如图所示64位的windows系统是Server模式,而其默认的收集器是parallel scavenge。
![不同系统下的server](/images/default_server.png)

为什么这个收集器就这么神奇呢？经过我的探索发现
```
The problem is os::page_size_for_region, in particular the check: 
if ((region_min_size & mask) == 0 && (region_max_size & mask) == 0) 

If page_size_for_region is called with min_region_size and max_region_size that are a multiple of a page size larger than than max_page_size, then page_size_for_region will return that page. 

This problem manifests itself on Solaris using large pages. Solaris will use a page size of 2 MB. If -Xmx4m is used, then GenerationSizer::initialize_size_info will call page_size_for_region(4MB, 4MB, 8 /* min_pages*/). Since 4MB is a multiple 2MB, page_size_for_region will return 2MB. The bug is that we ask for at least 8 pages, which would limit the page size to 4MB/8 = 0.5MB (512 KB). Since parallel scavenge needs at least 4 pages, we get a heap size 4 * 2MB = 8MB, even though the user specified -Xmx4m!
```

事情当然不会到了这里就结束。我们知道-XX:SurvivorRatio表示Eden区与Survivor区的大小比值，默认是8，也就是说eden:from=8，但是很明显我们打印出来的信息并不是这样,2048:320=6,那么这个8哪里来的呢？顿时我又崩溃了,还好经过我不管的查找资料，翻看JDK源码,发现了原来是是这样的
```
// The survivor ratio's are calculated "raw", unlike the
    // default gc, which adds 2 to the ratio value. We need to
    // make sure the values are valid before using them.
    if (MinSurvivorRatio < 3) {
      MinSurvivorRatio = 3;
    }
```
这里可以查看资料[parallelScavenge中GenerationSize](https://searchcode.com/codesearch/view/17980811/)

你以为到了这里问题就完了吗？我告诉你并没有。

### 解决一个问题
我遇到这个问题的时候我也是很崩溃的,我先是翻阅书籍，然后发现没有，然后为群友，去社区提问题，还是没有能够解决这个问题。

但是我是谁呀！我没有放弃，没有得过起过，我还和你耗上了。然后我去翻阅官方文档，然后让我发现了这个client和server端的区别，是的之前我没有发现这个。然后官方文档上没接着说这些细节，然后我就去找源码呀！我看人家源码怎么说的呀，这样一层一层找下来，最终问题解决了。

虽然现在看来我写出来这个问题真的很小，可是在找问题的过程中确实花费了很多事件，但是这也锻炼了我解决问题的思维。

我的方法如下：
1. 确定是自己配置不对还是自己理解有误
2. 在1的基础上定位问题解决方向是寻找官方文档还是加深理解
3. 找答案，做实验。

### 参考文档
1. http://docs.oracle.com/javase/7/docs/technotes/guides/vm/server-class.html
2. http://www.oracle.com/technetwork/java/javase/gc-tuning-6-140523.html
3. http://www.oracle.com/technetwork/java/faq-140837.html
4. https://bugs.openjdk.java.net/browse/JDK-4484370
5. https://bugs.openjdk.java.net/browse/JDK-8027915
6. https://searchcode.com/codesearch/view/17980811/