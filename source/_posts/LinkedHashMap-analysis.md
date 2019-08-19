---
title: LinkedHashMap源码解析
date: 2019-08-16 16:13:36
tags: java
---

上一篇文章分析了HashMap的原理,有网友留言想看LinkedHashMap分析，今天它来了。

LinkedHashMap是HashMap的子类,在原有HashMap数据结构的基础上,它还维护着一个双向链表链接所有entry,这个链表定义了迭代顺序，通常是数据插入的顺序。

![LinkedHashMap结构](/images/java/LinkedHashMap_structure.png)

<!--more-->

上图我只画了链表，其实红黑树节点也是一样的，只是节点类型不一样而已

也就是说我们遍历LinkedHashMap的时候,是从head指针指向的节点开始遍历,一直到tail指向的节点。

### 源码

```java
public class LinkedHashMap<K,V> extends HashMap<K,V> implements Map<K,V>
{
  static class Entry<K,V> extends HashMap.Node<K,V> {
    Entry<K,V> before, after;
    Entry(int hash, K key, V value, Node<K,V> next) {
        super(hash, key, value, next);
    }

    // 双向链表头节点
    transient LinkedHashMap.Entry<K,V> head;

    // 双向链表尾节点
    transient LinkedHashMap.Entry<K,V> tail;

    // 指定遍历LinkedHashMap的顺序,true表示按照访问顺序,false表示按照插入顺序，默认为false
    final boolean accessOrder;
  }

}
```
从LinkedHashMap的定义里面可以看到它单独维护了一个双向链表，用于记录元素插入的顺序。这也是为什么我们打印LinkedHashMap的时候可以按照插入顺序打印的支撑。而accessOrder属性则指明了进行遍历时是按照什么顺序进行访问,我们可以通过它的构造方法自己指定顺序。
```java
public LinkedHashMap(int initialCapacity,float loadFactor,
                   boolean accessOrder) {
  super(initialCapacity, loadFactor);
  this.accessOrder = accessOrder;
}
```
当accessOrder=true,访问顺序的输出是什么意思呢？来看下下面的一段代码
```java
LinkedHashMap<Integer,Integer> map = new LinkedHashMap<>(8, 0.75f, true);

map.put(1, 1);
map.put(2, 2);
map.put(3, 3);

map.get(2);

System.out.println(map);

```

输出结果是
```
{1=1, 3=3, 2=2}
```

可以看到get了的数据被放到了双向链表尾部，也就是按照了访问时间进行排序,这就是访问顺序的含义。


在插入的时候LinkedHashMap复写了HashMap的newNode以及newTreeNode方法,并且在方法内部更新了双向链表的指向关系。

同时插入的时候调用了afterNodeAccess()方法以及afterNodeInsertion()方法,在HashMap中这两个方法是空实现,而在LinkedHashMap中则有具体实现,这两个方法也是专门给LinkedHashMap进行回调处理的。

afterNodeAccess()方法中如果accessOrder=true时会移动节点到双向链表尾部。当我们在调用map.get()方法的时候如果accessOrder=true也会调用这个方法,这就是为什么访问顺序输出时访问到的元素移动到链表尾部的原因。

接下来来看看afterNodeInsertion()的实现

```java
// evict如果为false，则表处于创建模式,当我们new HashMap(Map map)的时候就处于创建模式
void afterNodeInsertion(boolean evict) { // possibly remove eldest
  LinkedHashMap.Entry<K,V> first;

  // removeEldestEntry 总是返回false,所以下面的代码不会执行。
  if (evict && (first = head) != null && removeEldestEntry(first)) {
      K key = first.key;
      removeNode(hash(key), key, null, false, true);
  }
}
```

看到这里我有一个想法,可以通过LinkedHashMap来实现LRU(Least Recently Used,即近期最少使用),只要满足条件,就删除head节点。

```
public class LRUCache<K,V> extends LinkedHashMap<K,V> {
    
  private int cacheSize;
  
  public LRUCache(int cacheSize) {
      super(16,0.75f,true);
      this.cacheSize = cacheSize;
  }

  /**
   * 判断元素个数是否超过缓存容量
   */
  @Override
  protected boolean removeEldestEntry(Map.Entry<K, V> eldest) {
      return size() > cacheSize;
  }
}

```
就这样一个简单的LRU Cache就实现了,以后面试官如果喊你给它实现一个LRU,你就这样写给他,如果他让你换一种方式,你就用链表使用同样的思维给他实现一个,然后你就可以收割offer了。


对于删除,LinkedHashMap也同样是在HashMap的删除逻辑完成后，调用了afterNodeRemoval这个回调方法来更正链表指向关系。


其实你只要看了上一篇文章再也不怕面试官问我JDK8 HashMap,再记得LinkedHashMap只是多维护了一个双向链表之后,再看LinkedHashMap中关于链表操作的代码就非常简单了。

