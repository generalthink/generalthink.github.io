---
title: 请别再问我Jdk8 HashMap了，好吗？
date: 2019-07-29 18:08:35
tags: java
---

上一篇文章中提到了ThreadLocalMap是使用开放地址法来解决冲突问题的，而我们今天的主角HashMap是采用了链表法来处理冲突的,什么是链表法呢?

![jdk8链表法](/images/java/hashmap-structure.png)

在散列表中，每个 “ 桶(bucket)” 或者 “ 槽(slot)” 会对应一条链表，所有散列值相同的元素我们都放到相同槽位对应的链表中。

jdk8和jdk7不一样，jdk7中没有红黑树,数组中只挂载链表。而jdk8中在桶容量大于等于64且链表节点数大于等于8的时候转换为红黑树。当红黑树节点数量小于6时又会转换为链表。

<!--more-->

### 插入

但插入的时候,我们只需要通过散列函数计算出对应的槽位,将其插入到对应链表或者红黑树即可。如果此时元素数量超过了一定值则会进行扩容，同时进行rehash.

### 查找或者删除

通过散列函数计算出对应的槽，然后遍历链表或者删除

### 链表为什么会转为红黑树?

上一篇文章有提到过通过装载因子来判定空闲槽位还有多少，如果超过装载因子的值就会动态扩容,HashMap会扩容为原来的两倍大小(初始容量为16,即槽(数组)的大小为16)。但是无论负载因子和散列函数设得再合理，也避免不了链表过长的情况，一旦链表过长查找和删除元素就比较耗时，影响HashMap性能,所以JDK8中对其进行了优化，当链表长度大于等于8的时候将链表转换为红黑树，利用红黑树的特点(查找、插入、删除的时间复杂度最坏为O(logn))，可以提高HashMap的性能。当节点个数少于6个的时候，又会将红黑树转化为链表。因为在数据量较小的情况下，红黑树要维持平衡，比起链表来，性能上的优势并不明显，而且编码难度比链表要大上不少。


### 源码分析

#### 构造方法以及重要属性

```java
public HashMap(int initialCapacity, float loadFactor);

public HashMap(int initialCapacity);

public HashMap();

```

HashMap的构造方法中可以分别指定初始化容量(bucket大小)以及负载因子，如果不指定默认值分别是16和0.75.它几个重要属性如下：

```java
// 初始化容量，必须要2的n次幂
static final int DEFAULT_INITIAL_CAPACITY = 1 << 4; // aka 16

// 负载因子默认值
static final float DEFAULT_LOAD_FACTOR = 0.75f;

// 需要从链表转换为红黑树时,链表节点的最小长度
static final int TREEIFY_THRESHOLD = 8;

// 转换为红黑树时数组的最小容量
static final int MIN_TREEIFY_CAPACITY = 64;

// resize操作时,红黑树节点个数小于6则转换为链表。
static final int UNTREEIFY_THRESHOLD = 6;

// HashMap阈值，用于判断是否需要扩容(threshold = 容量*loadFactor)
int threshold;

// 负载因子
final float loadFactor;

// 链表节点
static class Node<K,V> implements Map.Entry<K,V> {
  final int hash;
  final K key;
  V value;
  Node<K,V> next;

}

// 保存数据的数组
transient Node<K,V>[] table;

// 红黑树节点
static final class TreeNode<K,V> extends LinkedHashMap.Entry<K,V> {
  TreeNode<K,V> parent;  // red-black tree links
  TreeNode<K,V> left;
  TreeNode<K,V> right;
  TreeNode<K,V> prev;    // needed to unlink next upon deletion
  boolean red;
}
```
上面的table就是存储数据的数组(可以叫做桶或者槽),数组挂载的是链表或者红黑树。值得一提的是构造HashMap的时候并没有初始化数组容量，而是在第一次put元素的时候才进行初始化的。

### hash函数的设计

```java
int hash = (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
int index = hash & (tab.length-1);
```

从上面可以看出,key为null是时候放到数组中的第一个位置的,我们一般定位key应当存放在数组哪个位置的时候一般是这样做的 `key.hashCode() % tab.length`。但是当tab.length是2的n次幂的时候，就可以转换为 `A % B = A & (B-1)`;所以 `index = hash & (tab.length-1)`就可以理解了。
>这里是使用了除留余数法的理念来设计的,可以可能减少hash冲突
>除留余数法 : 用关键字K除以某个不大于hash表长度m的数p,将所得余数作为hash表地址
>比如x/8=x>>3,即把x右移3位，得到了x/8的商，被移掉的部分(后三位)，则是x%8，也就是余数。

而对于hash值的运算为什么是`(h = key.hashCode()) ^ (h >>> 16)`呢？也就是为什么要向右移16位呢?直接使用 `key.hashCode() & (tab.length -1)`不好吗？
如果这样做，由于tab.length肯定是远远小于hash值的,所以位运算的时候只有低位才参与运算，而高位毫无作为，会带来hash冲突的风险。

**而hashcode本身是一个32位整形值，向右移位16位之后再进行异或运行计算出来的整形将具有高位和低位的性质，就可以得到一个非常随机的hash值，在通过除留余数法，得到的index就更低概率的减少了冲突。**


### 插入数据

```java
final V putVal(int hash, K key, V value, boolean onlyIfAbsent,
                  boolean evict) {

 Node<K,V>[] tab; Node<K,V> p; int n, i;

 // 1. 如果数组未初始化,则初始化数组
 if ((tab = table) == null || (n = tab.length) == 0)
    n = (tab = resize()).length;

 // 2. 如果当前节点未被插入数据(未碰撞),则直接new一个节点进行插入
 if ((p = tab[i = (n - 1) & hash]) == null)
    tab[i] = newNode(hash, key, value, null);
 else {
    Node<K,V> e; K k;

    // 3. 碰撞了,已存在相同的key,则进行覆盖
   if (p.hash == hash &&
       ((k = p.key) == key || (key != null && key.equals(k))))
       e = p;
   else if (p instanceof TreeNode)
        // 4. 碰撞后发现为树结构，则挂载在树上
       e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
   else {
       for (int binCount = 0; ; ++binCount) {
            // 5. 进行尾插入,如果链表节点数达到上线则转换为红黑树
           if ((e = p.next) == null) {
               p.next = newNode(hash, key, value, null);
               if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                   treeifyBin(tab, hash);
               break;
           }
           // 6. 链表中碰撞了
           if (e.hash == hash &&
               ((k = e.key) == key || (key != null && key.equals(k))))
               break;
           p = e;
       }
     }
     // 7. 用新value替换旧的value
     if (e != null) { // existing mapping for key
       V oldValue = e.value;
       if (!onlyIfAbsent || oldValue == null)
           e.value = value;
       afterNodeAccess(e);
       return oldValue;
     }
 }
 ++modCount;

 // 8. 操作阈值则进行扩容
 if (++size > threshold)
     resize();

 // 给LinkedHashMap实现
 afterNodeInsertion(evict);
 return null;
}
```

简述下put的逻辑，它主要分为以下几个步骤:

1. 首先判断是否初始化，如果未初始化则初始化数组,初始容量为16
2. 通过hash&(n-1)获取数组下标，如果该位置为空，表示未碰撞，直接插入数据
3. 发生碰撞且存在相同的key，则在后面处理中直接进行覆盖
4. 碰撞后发现为树结构，则直接挂载到红黑树上
5. 碰撞后发现为链表结构，则进行尾插入，当链表容量大于等于8的时候转换为树节点
6. 发现在链表中进行碰撞了，则在后面处理直接覆盖
7. 发现之前存在相同的key,只直接用新值替换旧值
8. map的容量(存储元素的数量)大于阈值则进行扩容，扩容为之前容量的2倍


### 扩容

resize()方法中，如果发现当前数组未初始化，则会初始化数组。如果已经初始化，则会将数组容量扩容为之前的两倍，同时进行rehash(将旧数组的数据移动到新的数组).JDK8的rehash过程很有趣，相比JDK7做了不少优化，我们来看下这里的rehash过程。

```java

// 数组扩容为之前2倍大小的代码省略，这里主要分析rehash过程。

if (oldTab != null) {
 // 遍历旧数组
 for (int j = 0; j < oldCap; ++j) {
   Node<K,V> e;
   if ((e = oldTab[j]) != null) {
     oldTab[j] = null;

     // 1. 如果旧数组中不存在碰撞,则直接移动到新数组的位置
     if (e.next == null)
        newTab[e.hash & (newCap - 1)] = e;
     else if (e instanceof TreeNode)
        // 2. 如果存在碰撞，且节点类型是树节点，则进行树节点拆分(挂载到扩容后的数组中或者转为链表)
        ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
     else { // preserve order

        // 3. 处理冲突是链表的情况,会保留原有节点的顺序

       Node<K,V> loHead = null, loTail = null;
       Node<K,V> hiHead = null, hiTail = null;
       Node<K,V> next;
       do {
         next = e.next;
         // 4. 判断扩容后元素是否在原有的位置(这里非常巧妙,下面会分析)
         if ((e.hash & oldCap) == 0) {
           if (loTail == null)
               loHead = e;
           else
               loTail.next = e;
           loTail = e;
         }

         // 5. 元素不是在原有位置
         else {
           if (hiTail == null)
               hiHead = e;
           else
               hiTail.next = e;
           hiTail = e;
         }
       } while ((e = next) != null);

       // 6. 将扩容后未改变index的元素复制到新数组
       if (loTail != null) {
         loTail.next = null;
         newTab[j] = loHead;
       }

       // 7. 将扩容后改变了index位置的元素复制到新数组
       if (hiTail != null) {
         hiTail.next = null;
         // 8. index改变后,新的下标是j+oldCap,这里也很巧妙，下面会分析
         newTab[j + oldCap] = hiHead;
       }
     }
   }
 }
}
```
上面的代码中展现了整个rehash的过程，先遍历旧数组中的元素，接着做下面的事情

1. 如果旧数组中不存在数据碰撞(未挂载链表或者红黑树),那么直接将元素赋值到新数组中，其中`index=e.hash & (newCap - 1)`。
2. 如果存在碰撞，且节点类型是树节点，则进行树节点拆分(挂载到扩容后的数组中或者转为链表)
3. 如果存在碰撞，且节点是链表，则处理链表的情况,**rehash过程会保留节点原始顺序(JDK7中不会保留，这也是导致jdk7中多线程出现死循环的原因)**
4. 判断元素在扩容后是否还处于原有的位置，这里通过`(e.hash & oldCap) == 0`判断,oldCap表示扩容前数组的大小。
5. 发现元素不是在原有位置，更新hiTail和hiHead的指向关系
6. 将扩容后未改变index的元素复制到新数组
7. 将扩容后改变了index位置的元素复制到型数组，新数组的下标是 `j + oldCap`。

其中第4点和第5点中将链表的元素分为两部分(do..while部分)，一部分是rehash后index未改变的元素，一部分是index被改变的元素。分别用两个指针来指向头尾节点。
>比如当oldCap=8时,1-->9-->17都挂载在tab[1]上,而扩容后，1-->17挂载在tab[1]上,9挂载在tab[9]上。

那么是如何确定rehash后index是否被改变呢？改变之后的index又变成了多少呢？

这里的设计很是巧妙，还记得HashMap中数组大小是2的n次幂吗?当我们计算索引位置的时候，使用的是 e.hash & (tab.length -1)。

这里我们讨论数组大小从8扩容到16的过程。
```
tab.length -1 = 7   0 0 1 1 1
e.hashCode = x      0 x x x x
==============================
                    0 0 y y y  

```
可以发现在扩容前index的位置由hashCode的低三位来决定。那么扩容后呢？

```
tab.length -1 = 15   0 1 1 1 1
e.hashCode = x       x x x x x
==============================
                     0 z y y y

```

扩容后，index的位置由低四位来决定,而低三位和扩容前一致。也就是说扩容后index的位置是否改变是由高字节来决定的,也就是说我们只需要将hashCode和高位进行运算即可得到index是否改变。

而刚好扩容之后的高位和oldCap的高位一样。如上面的15二进制是1111,而8的二进制是1000,他们的高位都是一样的。所以我们通过e.hash & oldCap运算的结果即可判断index是否改变。

同理，如果扩容后index该变了。新的index和旧的index的值也是高位不同，其新值刚好是 oldIndex + oldCap的值。所以当index改变后,新的index是 j + oldCap。


至此,resize方法结束,元素被插入到了该有的位置。

### get()

get()的方法就相对来说要简单一些了，它最重要的就是找到key是存放在哪个位置

```java
final Node<K,V> getNode(int hash, Object key) {
  Node<K,V>[] tab; Node<K,V> first, e; int n; K k;

  // 1. 首先(n-1) & hash确定元素位置
  if ((tab = table) != null && (n = tab.length) > 0 &&
      (first = tab[(n - 1) & hash]) != null) {

      // 2. 判断第一个元素是否是我们需要找的元素
      if (first.hash == hash &&
          ((k = first.key) == key || (key != null && key.equals(k))))
          return first;
      if ((e = first.next) != null) {
        // 3. 节点如果是树节点,则在红黑树中寻找元素
        if (first instanceof TreeNode)
            return ((TreeNode<K,V>)first).getTreeNode(hash, key);
        4. 在链表中寻找对应的节点
        do {
            if (e.hash == hash &&
                ((k = e.key) == key || (key != null && key.equals(k))))
                return e;
        } while ((e = e.next) != null);
      }
  }
  return null;
}

```

### remove

remove方法寻找节点的过程和get()方法寻找节点的过程是一样的，这里我们主要分析寻找到节点后是如何处理的

```java
if (node != null && (!matchValue || (v = node.value) == value ||
    (value != null && value.equals(v)))) {
    // 1. 删除树节点,删除时如果不平衡会重新移动节点位置
    if (node instanceof TreeNode)
        ((TreeNode<K,V>)node).removeTreeNode(this, tab, movable);
    // 删除的节点是链表第一个节点,则直接将第二个节点赋值为第一个节点
    else if (node == p)
        tab[index] = node.next;
    // 删除的节点是链表的中间节点，这里的p为node的prev节点
    else
        p.next = node.next;
    ++modCount;
    --size;
    afterNodeRemoval(node);
    return node;
}

```
remove方法中，最为复杂的部分应该是removeTreeNode部分，因为删除红黑树节点后，可能需要退化为链表节点，还可能由于不满足红黑树特点，需要移动节点位置。
代码也比较多，这里就不贴上来了。但也因此佐证了为什么不全部使用红黑树来代替链表。


### JDK7扩容时导致的死循环问题

```java
/**
* Transfers all entries from current table to newTable.
*/
void transfer(Entry[] newTable) {
 Entry[] src = table;
 int newCapacity = newTable.length;
 for (int j = 0; j < src.length; j++) {
   Entry<K,V> e = src[j];
   if (e != null) {
       src[j] = null;
       do {
           // B线程执行到这里之后就暂停了
           Entry<K,V> next = e.next;
           int i = indexFor(e.hash, newCapacity);
           e.next = newTable[i];
           newTable[i] = e;
           e = next;
       } while (e != null);
   }
 }
}

```
扩容时上面的代码容易导致死循环,是怎样导致的呢？假设有两个线程A和B都在执行这一段代码，数组大小由2扩容到4,在扩容前tab[1]=1-->5-->9。
![扩容前](/images/java/hashmap-unresize.png)

当B线程执行到 next = e.next时让出时间片,A线程执行完整段代码但是还没有将内部的table设置为新的newTable时，线程B继续执行。

此时A线程执行完成之后，挂载在tab[1]的元素是9-->5-->1,注意这里的**顺序被颠倒了**。此时e = 1, next = 5;
>tab[i]的按照循环次数变更顺序, 1. tab[i]=1, 2. tab[i]=5-->1, 3. tab[i]=9-->5-->1

![线程A执行完成后](/images/java/hashmap-resize-threadA.png)

同样B线程我们也按照循环次数来分析
1. 第一次循环执行完成后,newTable[i]=1, e = 5
2. 第二次循环完成后: newTable[i]=5-->1, e = 1。 
3. 第三次循环,e没有next,所以next指向null。当执行e.next = newTable[i]\(1-->5)的时候,就形成了 1-->5-->1的环,再执行newTable[i]=e,此时newTable[i] = 1-->5-->1。

当在数组该位置get寻找对应的key的时候，就发生了死循环,引起CPU 100%问题。

![线程B执行扩容过程](/images/java/hashmap-resize.png)


而JDK8就不会出现这个问题,它在这里就有一个优化，它使用了两个指针来分别指向头结点和尾节点，而且还保证了元素原本的顺序。
当然HashMap仍然是不安全地,所以在多线程并发条件下推荐使用ConcurrentHashMap。