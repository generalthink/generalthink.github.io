---
title: 从字节码角度来看try-catch-finally是如何执行的
date: 2020-09-01 22:25:10
tags: java
---

三年前，我做了一道关于try-catch-finnaly的面试题，但我做错了，当时面试官问我为啥错了，我告诉它，我平常不会写这么傻逼的代码，然后面试官就没有问我了。。。。

最近看到其他面试的童鞋，又让我想起了这道题，刚好也试着分析下。

我们知道Java虚拟机栈是线程私有的，它的生命周期与线程相同。虚拟机栈是Java虚拟机运行时数据区一部分，它描述的是Java方法执行的内存模型：每个方法在执行的同时都会创建一个栈帧用于存储局部变量表、操作数栈、动态链接、方法出口等信息。**每一个方法从调用直至执行完成的过程就对应着一个帧栈在虚拟机中入栈到出栈的过程。**

<!--more-->

加载和存储指令用于将数据在帧栈中的局部变量和操作数栈之前来回传输，这类指令包括如下内容：

1. 将一个局部变量加载到操作栈：iload， iload_\<n\>、aload、aload_\<n\>等

2. 将一个数值从操作数栈存储到局部变量表： istore、 istore_\<n\>、astore、astore_\<n\>等

3. 将一个常量加载到操作数栈： ldc、iconst_\<i\>等


比如我们下面的代码

```java
public class Test {

  public static void main(String[] args) {

      System.out.println(getNumber());
  }

  public static int getNumber() {
    int x;
    try {
      x=1;
      return x;
    } catch (Exception e) {
      x=2;
      return x;
    }
    finally{
      x=3;
    }

  }

}

```

上面的代码输出结果是1，我们首先来看看它的字节码是怎样的？

```
 public static int getNumber();
  Code:
   0: iconst_1
   1: istore_0
   2: iload_0
   3: istore_1
   4: iconst_3
   5: istore_0
   6: iload_1
   7: ireturn
   8: astore_1
   9: iconst_2
  10: istore_0
  11: iload_0
  12: istore_2
  13: iconst_3
  14: istore_0
  15: iload_2
  16: ireturn
  17: astore_3
  18: iconst_3
  19: istore_0
  20: aload_3
  21: athrow
Exception table:
   from    to  target type
       0     4     8   Class java/lang/Exception
       0     4    17   any
       8    13    17   any
  }
```

我来根据字节码来分析下是代码是如何运行的:

操作数栈: 先进后出的一个数据结构
局部变量表: 可以认为是一个数组，下标从0开始

以下是执行每一条指令的时候操作数栈和局部变量表的变化情况：

```
iconst_1(将int类型数字1放入操作数栈顶):
操作数栈: 1
局部变量表：

istore_0(将操作数栈顶int型数字出栈存入变量表第1个本地变量):
操作数栈: 
局部变量表：1

iload_0(将变量表第1个int型本地变量推送至栈顶):
操作数栈: 1
局部变量表:

istore_1(将操作数栈顶int型数字出栈存入变量表第2个本地变量):
操作数栈:
局部变量表: null 1

iconst_3(将int类型数字3放入操作数栈顶):
操作数栈: 3
局部变量表: null 1

istore_0(将操作数栈顶int型数字出栈存入变量表第1个本地变量):
操作数栈: 
局部变量表: 3 1

iload1(将变量表第2个int型本地变量推送至栈顶):
操作数栈: 1
局部变量表: 3

ireturn(从栈顶返回int型数字，方法结束):
返回1

```

可以看到ireturn后面的代码就不会被执行了，我们也就不进行翻译了。**实际上try-catch-finally字节码块中是没有finally的，**根据字节码我们可以将代码简化成这样

```java
public static int getNumber() {
  int x;
  int returnValue;
  try {
    x=1;
    returnValue = x;
    x = 3;
    return returnValue;
  } catch (Exception e) {
    x=2;
    return x;
  }
}
```

那如果将代码变成这样呢？

```java
public static int getNumber() {
  int x;
  try {
    x=1;
    return x;
  } catch (Exception e) {
    x=2;
    return x;
  }
  finally{
    x=3;
    return x;
  }
}

```
我相信你能知道输出的结果是3，我们同样来看下字节码是怎样的？

```
public static int getNumber();
  Code:
     0: iconst_1
     1: istore_0
     2: iload_0
     3: istore_1
     4: iconst_3
     5: istore_0
     6: iload_0
     7: ireturn
     8: astore_1
     9: iconst_2
    10: istore_0
    11: iload_0
    12: istore_2
    13: iconst_3
    14: istore_0
    15: iload_0
    16: ireturn
    17: astore_3
    18: iconst_3
    19: istore_0
    20: iload_0
    21: ireturn
  Exception table:
     from    to  target type
         0     4     8   Class java/lang/Exception
         0     4    17   any
         8    13    17   any
}

```

同样的我们只分析执行到的部分

```
iconst_1(将int类型数字1放入操作数栈顶):
操作数栈: 1
局部变量表：

istore_0(将操作数栈顶int型数字出栈存入变量表第1个本地变量):
操作数栈: 
局部变量表：1

iload_0(将变量表第1个int型本地变量推送至栈顶):
操作数栈: 1
局部变量表：

istore_1(将操作数栈顶int型数字出栈存入变量表第2个本地变量):
操作数栈: 
局部变量表： null 1

iconst_3(将int类型数字3放入操作数栈顶):
操作数栈: 3
局部变量表：null 1

istore_0(将操作数栈顶int型数字出栈存入变量表第1个本地变量):
操作数栈: 
局部变量表：3 1

# 注意这里和上面字节码的不同之处在于上面是加载变量表中的第二个int类型本地变量
iload_0(将变量表第1个int型本地变量推送至栈顶):
操作数栈: 3
局部变量表：1

ireturn(从栈顶返回int型数字，方法结束):
返回3

```

上面都只是分析了try-finally，我们接着分析下try-catch-finally是如何的

```java

public static int getNumber() {
  int x;
  try {
    x = 1/0;
    return x;
  } catch (Exception e) {
    x = 2;
    return x;
  } finally {
    x = 3;
    return x;
  }
}
```

之前的代码很明显不会抛出异常，所以就用不到异常表中的内容，但是这里肯定是产生异常(1/0)，而在java中对异常的处理在字节码层面是使用Exception Table来完成的。

```
public static int getNumber();
Code:
 0: iconst_1
 1: iconst_0
 2: idiv
 3: istore_0
 4: iload_0
 5: istore_1
 6: iconst_3
 7: istore_0
 8: iload_0
 9: ireturn
10: astore_1
11: iconst_2
12: istore_0
13: iload_0
14: istore_2
15: iconst_3
16: istore_0
17: iload_0
18: ireturn
19: astore_3
20: iconst_3
21: istore_0
22: iload_0
23: ireturn
Exception table:
 from    to  target type
     0     6    10   Class java/lang/Exception
     0     6    19   any
    10    15    19   any
}

```

这个异常表(Exception table)含义是如果当字节码在第from行到第to行之间（不包含to行）出现了类型为type或者其子类的异常则转到第target行继续处理。当type的值为any时，代表任意异常情况都需要转向到target处进行处理。

异常表也相当于指明了代码有可能的分支执行情况。老规矩，我们一行一行来分析字节码

```
iconst_1(将int类型数字1放入操作数栈顶):
操作数栈: 1
局部变量表：

iconst_0(将int类型数字0放入操作数栈顶):
操作数栈: 0 1
局部变量表：

idiv(将操作数栈顶两int型数值相除，并将结果压入栈顶):
操作数栈:
局部变量表:

经过上面的操作(1/0)抛出异常，此时根据异常表，执行第10行的字节码

astore_1(将栈顶引用型数字存入变量表第2个本地变量，因为栈顶为空，所以都为空):
操作数栈:
局部变量表:

iconst_2(将int类型数字2放入操作数栈顶)
操作数栈: 2
局部变量表：


istore_0(将操作数栈顶int型数字出栈存入变量表第1个本地变量):
操作数栈: 
局部变量表：2


iload_0(将变量表第1个int型本地变量推送至栈顶):
操作数栈: 2
局部变量表：

istore_2(将操作数栈顶int型数字出栈存入变量表第1个本地变量):
操作数栈: 
局部变量表：2

iconst_3(将int类型数字3放入操作数栈顶):
操作数栈: 3
局部变量表：2

istore_0(将操作数栈顶int型数字出栈存入变量表第1个本地变量):
操作数栈: 
局部变量表：3

iload_0(将变量表第1个int型本地变量推送至栈顶):
操作数栈: 3
局部变量表：

ireturn(从栈顶返回int型数字，方法结束):
返回3

```

经过上面的分析我们可以知道，try-catch-finally就是很普通的指令跳转而已，我们最需要记住的是当return的时候，实际上并不会马上return，而是会将这个结果存入这个临时变量，然后再返回这个临时变量。由于本文所举例的代码中使用的是基本类型，所以对值的修改看上去没有起作用，但是如果“i”是对可变类对象的引用，并且对象的内容在finally块中进行了更改，则这些更改也将反映在返回的值中。