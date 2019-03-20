---
title: 从Java字节码分析中我学到了什么
date: 2019-03-19 13:11:58
tags: java
---

### 实参和形参

```java
public int sum(int x,int y) {
	return x+y;
}

sum(2,3);
```
上面的代码中sum()方法中的x,y就是形参,而调用方法sum(2,3)中的2与3就是实参。形参是在方法定义阶段，而实参实在方法调用阶段。
<!--more-->

### 查看字节码

#### 基本类型参数调用
```java
private static int intStatic = 222;

public static void main(String[] args) {
    method(intStatic);
    System.out.println(intStatic);
}

public static void method(int intStatic) {
    intStatic = 777;
}

```

上面的method()方法的字节码(`javap -verbose XXX.class`)如下:
```
public static void method(int);
  descriptor: (I)V
  flags: ACC_PUBLIC, ACC_STATIC
  Code:
    stack=1, locals=1, args_size=1
       0: sipush        777
       3: istore_0
       4: return
    LineNumberTable:
      line 13: 0
      line 14: 4
    LocalVariableTable:
      Start  Length  Slot  Name   Signature
          0       5     0 intStatic   I
```

>sipush: 将一个短整形常量值推送至栈顶
>>iconst 将int形式-1-5推送到栈顶
>>bipush表示将单字节的常量值(-128-127)推送至栈顶
>>sipush表示一个短整形常量值(-32768-32767)推送至栈顶

>istore_0: 将栈顶int型数值存入第一个本地变量。

上面字节码的意思就是将777推送到栈顶，然后将其赋值给intStatic这个本地变量。所以我们输出的结果是222,因为method()中的赋值是对本地变量进行赋值的,并没有改变static变量的值,这也是Java的变量就近原则,当然可以使用Class.intStatic这样的方式显示声明。


#### 不可变对象参数调用

```java
private static String stringStatic = "old string";

public static void main(String[] args) {
    method(stringStatic);
    System.out.println(stringStatic);
}
public static void method(String stringStatic) {
    stringStatic = "new string";
}

```
上面输出的结果是`old string`,同样反编译看下字节码


```
Constant pool:
  #6 = String     #35          // new string
  #7 = String     #36          // old string
  #10 = Utf8      stringStatic
  #35 = Utf8      new string
  #36 = Utf8      old string

public static void method(java.lang.String);
  descriptor: (Ljava/lang/String;)V
  flags: ACC_PUBLIC, ACC_STATIC
  Code:
    stack=1, locals=1, args_size=1
       0: ldc           #6       // String new string
       2: astore_0
       3: return
    LocalVariableTable:
        Start  Length  Slot     Name             Signature
          0      4      0    stringStatic   Ljava/lang/String;
```
>ldc: 将int、float或String型常量值从常量池中推送至栈顶。
>astore_0: 将栈顶引用型数值存入第一个本地变量。

字节码的意思是将#6(符号引用)的值推送至栈顶,然后将其引用型数值赋值给第一个本地变量(stringStatic)。

可以看到这里的#6表示是一个String类型数据，它指向常量池中一个CONSTANT_Utf8_info(缩写Utf8,Class文件中方法字段等都需要引用它来描述名称)类型，这个常量代表了类(或者接口)的全限定名称,在运行的时候，JVM会根据这个全限定名称来实例化这个类,那个时候符号引用会被转换为直接引用(就是内存中的地址)。


#### 运行时常量池
上面一直在说常量池,那么常量池到底是个什么东东?

**我们都知道方法区与Java堆一样,是各个线程共享的内存区域，它用于存储已被虚拟机加载的类信息、常量、静态变量、即时编译器编译后的代码。
Class文件中除了有类的版本、字段、方法、接口等描述信息外，还有一项是常量池(Constant Pool Table)，用于存放编译期产生的各种字面量和符号引用，这部分内容将在类加载后进入方法区的运行时常量池中存放。
运行时常量池相对于Class文件常量池的另外一个重要特征是具备动态性，Java语言并不要求常量一定只有在编译器才能产生，也就是并非预置入Class文件常量池的内容才能进入方法区运行时常量池，运行期间也可以将新的常量放入池中,String.intern()方法就是典型的代表。**

>永久代和方法区，很多人会迷惑。方法区是JVM规范,而永久代只是实现。
>从JDK7开始常量池已经从方法区移动到堆中

文本字符串、声明为final的常量值都为字面量，而符号引用包含了下面三类常量
	1. 类和接口的全限定名
	2. 字段的名称和描述符
	3. 方法的名称和描述符

```
Constant pool:
  #1 = Methodref    #9.#28     // java/lang/Object."<init>":()V
  #2 = Fieldref     #8.#29     // com/generalthink/kafka/ParamDemo.stringStatic:Ljava/lang/String;
  #3 = Methodref    #8.#30     // com/generalthink/kafka/ParamDemo.method:(Ljava/lang/String;)V
  #8 = Class        #37        // com/generalthink/kafka/ParamDemo
```
所以,编译器**符号引用stringStatic**和**字面量old string**会被加入到Class文件的常量池中，然后在类加载阶段，这两个常量会进入运行时常量池。

#### 可变对象参数调用

上面的参数传递的是不可变对象，这里变成可变对象我们再次分析下

```java
private static StringBuilder stringBuilderStatic = new StringBuilder("old stringBuilder");

public static void main(String[] args) {
    method(stringBuilderStatic);
    System.out.println(stringBuilderStatic);
}

public static void method(StringBuilder stringBuilderStaticParam) {
    stringBuilderStaticParam.append(" first append");

    stringBuilderStaticParam = new StringBuilder("new stringBuilder");
    stringBuilderStaticParam.append(" new method's append");
}
```

查看对应的关键字节码如下:

```
Constant pool:
  #6 = String     #41            //  first append
  #8 = Class      #43            // java/lang/StringBuilder
  #9 = String     #44            // new stringBuilder
  #11 = String    #46            // new method's append
  #12 = String    #47            // old stringBuilder
  #15 = Utf8      stringBuilderStatic
  #30 = Utf8      stringBuilderStaticParam
  #41 = Utf8      first append
  #43 = Utf8      java/lang/StringBuilder
  #44 = Utf8      new stringBuilder
  #46 = Utf8      new method's append
  #47 = Utf8      old stringBuilder

public static void method(java.lang.StringBuilder);
  descriptor: (Ljava/lang/StringBuilder;)V
  flags: ACC_PUBLIC, ACC_STATIC
  Code:
    stack=3, locals=1, args_size=1
       0: aload_0
       1: ldc           #6    // String  first append
       3: invokevirtual #7    // Method java/lang/StringBuilder.append:(Ljava/lang/String;)Ljava/lang/StringBuilder;
       6: pop
       7: new           #8    // class java/lang/StringBuilder
      10: dup
      11: ldc           #9    // String new stringBuilder
      13: invokespecial #10   // Method java/lang/StringBuilder."<init>":(Ljava/lang/String;)V
      16: astore_0
      17: aload_0
      18: ldc           #11   // String new method's append
      20: invokevirtual #7    // Method java/lang/StringBuilder.append:(Ljava/lang/String;)Ljava/lang/StringBuilder;
      23: pop
      24: return
    LocalVariableTable:
      Start  Length  	Slot  		Name   						Signature
        0      25      0 		stringBuilderStaticParam   Ljava/lang/StringBuilder;
```
>aload_0:将第一个引用类型本地变量推送至栈顶。
>ldc : 将int、float或String型常量值从常量池中推送至栈顶。
>invokevirtual：用于调用一些需要特殊处理的实例方法，包括实例初始化方法、私有方法和父类方法
>pop: 将栈顶数值弹出(不能是long或double类型)
>new: 创建一个对象,并将其引用值压入栈顶
>dup：复制栈顶数值并将复制值压入栈顶
>astore_0 : 将栈顶引用型数值存入第一个本地变量
>return : 从当前方法返回void

需要注意的是aload_0中的0指的是`LocalVariableTable`中slot为0的参数，这里指的是stringBuilderStaticParam。它是把静态变量的引用赋值给虚拟机栈帧中的局部变量表。

上面字节码的意思就是stringBuilderStaticParam推送至栈顶,然后将first append常量值推送到栈顶，调用StringBuilder.append方法得到结果,最后出栈。
接着new一个StringBuilder,将返回的地址复制一份压入栈顶,然后在将这个地址存入stringBuilderStaticParam。然后重新aload到操作栈顶(这里的值已经被进行了覆盖,所以后续对于stringBuilderStaticParam的append操作与类的静态变量stringBuilderStatic没有任何关系),然后接着调用append方法,最后返回void。

需要注意的是stringBuilderStatic仅仅只是一个指针,一个指向内存中具体地址的指针而已,它并不是这个内存地址。java spec中声明说,**java中的所有东西都是值传递**,从没有引用传递这个玩意儿
代码是检验整理的唯一标准,现在假设是引用传递,那么执行method方法之后,输出的结果就应该是`new stringBuilder new method's append`,但是输出结果并不是，所以参数传递是值传递,这个值对对象来说是指针而已。

### this是如何实现的

```java
int m = 1;
public static void main(String[] args) {
    ParamDemo demo = new ParamDemo();
    System.out.println(demo.method());
}
public  int method() {
    return m + 1;
}
```

我们在method()方法中调用了m,这里隐式的使用了this,其实是this.m。那么this式如何实现的呢？

```
public int method();
  descriptor: ()I
  flags: ACC_PUBLIC
  Code:
    stack=2, locals=1, args_size=1
       0: aload_0
       1: getfield      #2     // Field m:I
       4: iconst_1
       5: iadd
       6: ireturn
    LineNumberTable:
      line 15: 0
    LocalVariableTable:
      Start  Length  Slot  Name   Signature
          0       7     0  this   Lcom/generalthink/kafka/ParamDemo;
}
```
我们在**LocalVariableTable**中可以发现this成为了一个参数，它的实现就是这样简单,**Javac编译器编译的时候把this关键字的访问转变为对一个普通方法参数的访问,然后在虚拟机调用实例方法时自动传入此参数。**

一说到this肯定就会想起super,其实super就是一个普通的方法调用,通过invokespecial指令实现。

### String.intern()原理

当使用intern()方法的时候你要想到方法作用是将字面量动态的加入运行时常量池。如果运行时常量池中已经存在了相同的字符串(equals方法决定),则返回池中的对象，否则将其加入到常量池后返回对应的引用。

```java
String s1 = "Hello";
String s2 = new String("Hello").intern();

//true
System.out.println(s1 == s2);
```
输出结果为true,编译期间"Hello"这个字面量已经加入到了常量池,运行期间,调用了intern()方法,先根据equals方法判断两个字符串相等,然后返回常量池当中Hello的引用地址,所以此时s1,s2其实指向的是同一个地址。

我们将代码做一点修改
```java
String s1 = "Hello";
String s2 = "World";

String s3 = s1 + s2;

String s4 = "Hello" + "World";

System.out.println(s3 == s4);

System.out.println(s3.intern() == s4);

```

输出结果分别为false和true。我们注意到s3和s4最多的不同就是,s3是由变量相加得到的,同样查看字节码

```
#2 = String     #37     // Hello
#3 = String     #38     // World
#4 = Class      #39     // java/lang/StringBuilder
#5 = Methodref  #4.#36  // java/lang/StringBuilder."<init>":()V
#6 = Methodref  #4.#40  // java/lang/StringBuilder.append:(Ljava/lang/String;)Ljava/lang/StringBuilder;
#7 = Methodref  #4.#41  // java/lang/StringBuilder.toString:()Ljava/lang/String;
#8 = String     #42     // HelloWorld

public static void main(java.lang.String[]);
  descriptor: ([Ljava/lang/String;)V
  flags: ACC_PUBLIC, ACC_STATIC
  Code:
    stack=3, locals=5, args_size=1
      0: ldc           #2        // String Hello
      2: astore_1
      3: ldc           #3        // String World
      5: astore_2
      6: new           #4        // class java/lang/StringBuilder
      9: dup
      10: invokespecial #5        // Method java/lang/StringBuilder."<init>":()V
      13: aload_1
      14: invokevirtual #6        // Method java/lang/StringBuilder.append:(Ljava/lang/String;)Ljava/lang/StringBuilder;
      17: aload_2
      18: invokevirtual #6        // Method java/lang/StringBuilder.append:(Ljava/lang/String;)Ljava/lang/StringBuilder;
      21: invokevirtual #7        // Method java/lang/StringBuilder.toString:()Ljava/lang/String;
      24: astore_3
      25: ldc           #8        // String HelloWorld
      27: astore        4
      ...
```
从字节码可以看出来s3实际上是调用了StringBuilder.append()方法来得到的,而s4的值在编译的时候就可以直接确定,它是一个准确的值,所以此时HelloWorld在常量池中就存在了。