title: 谈谈Java中的基本数据类型
date: 2016-07-18 08:19:16
tags: java
categories: java
description:
keywords: 基本数据类型 装箱 数据转换 基本数据类型和封装类型比较
---

### 认识Java中的基本数据类型

Java中有8大基本数据类型,它的对应包装类型以及数据范围如下:


|基本数据类型名称 |	封装数据类型名称 |	所占字节数	|	取值范围	|
|:--------|:----------------|:--------|:----------|
|boolean	|	Boolean	|	1	|	true/false	|
|byte	| 	Byte	|	1	|	-128~127	|
|char	|	Character	|	2	|	0~65535	|
|short	|	Short	|	2	|	-32768~32767	|
|int	|	Integer	|	4	|	-2的31次方到2的31次方-1	|
|long	|	Long	|	8	|	2的63次方到2的63次方-1	|
|float	|	Float	|	4	|	3.402823e+38 ~ 1.401298e-45  |
|double	|	Double	|	8	|	1.797693e+308~ 4.9000000e-324  |

**具体的字节数可以通过Character.SIZE,Double.SIZE等方法进行查看所占bit位,除以8就是所占字节数。**

值得一提的是Float.MIN\_VALUE以及Double.MIN\_VALUE以表示的是Float以及Double所能表示的最小正数，也就是说存在这样一种情况，0到±Float.MIN\_VALUE之间的值float类型无法表示，0 到±Double.MIN\_VALUE之间的值double类型无法表示。这并没有什么好奇怪的，因为这些范围内的数值超出了它们的精度范围。


### 装箱和拆箱
在编译阶段，若将原始类型int赋值给Integer类型，就会将原始类型自动编译为Integer.valueOf(int)，这种叫做装箱；如果将Integer类型赋值给int类型，则会自动转换调用intValue()方法，这种叫做拆箱。

我们来看一段代码：

```java
   Integer i = new  Integer(42);
   Integer j = new  Integer(42);
   System.out.println(i > j ? 1 : i == j ? 0 : -1);//-1
```

返回的是-1这是为什么呢? 执行i>j的时候会导致自动拆箱,也就是说直接比较它们的基本类型值,比较的结果肯定是否定的,那么接下来就会比较i==j,这里比较的是对象的引用,返回当然为false,所以最终结果为-1.

修改方案1:

```
int i2 = i;
int j2 = j;
System.out.println(i2 > j2 ? 1 : i2 == j2 ? 0 : -1);
```

修改方案2：

```
System.out.println(i > j ? 1 : i.intValue() == j.intValue() ? 0 : -1);
```

### 基本数据类型和封装类型的比较

先来看一段代码,关于基本数据类型和封装类型之间的比较

```
public static void main(String[] args) {
    Integer i = 127;
    Integer i2 = 127;
    
    int i3 = 127;
    
    Integer j = 128;
    Integer j2 = 128;
    int j3 = 128;
    
    Integer k = new Integer(127);
    Integer k2 = 127;
    
    Integer m = -128;
    int m2 = -128;
    
    Integer q = -129;
    Integer q2 = -129;
    int q3 = -129;
    Integer q4 = q3;

    System.out.println(i == i2);//true
    System.out.println(i == i3);//true
    System.out.println(j == j2);//false
    System.out.println(j == j3);//true
    System.out.println(k == k2);//false
    System.out.println(k == i3);//true
    System.out.println(m == m2);//true
    System.out.println(q == q2);//false
    System.out.println(q == q3);//true
    System.out.println(q == q4);//false
  }

```

从上面的结果可以看出了当Integer和int比较的时候，实际上是在做拆箱操作,当Integer和
Integer进行比较的时候会发现当值的范围在-128到127之间比较的时候总是为true(k==k2这种比较特殊,因为k是new的一个对象),那么为什么会出现当值在-128在127之间的时候比较总会相等呢？是因为Integer这个类对这个区间的数据做了一个缓存，具体可以查看JDK源码:

```

private static class IntegerCache {
	static final int low = -128;
	static final int high;
	static final Integer cache[];

	static {
		// high value may be configured by property
		int h = 127;
		String integerCacheHighPropValue =
		sun.misc.VM.getSavedProperty("java.lang.Integer.IntegerCache.high");
		if (integerCacheHighPropValue != null) {
			int i = parseInt(integerCacheHighPropValue);
			i = Math.max(i, 127);
			// Maximum array size is Integer.MAX_VALUE
			h = Math.min(i, Integer.MAX_VALUE - (-low));
		}
		high = h;

		cache = new Integer[(high - low) + 1];
		int j = low;
		for(int k = 0; k < cache.length; k++)
		cache[k] = new Integer(j++);
	}

	private IntegerCache() {}
	}

public static Integer valueOf(int i) {
	assert IntegerCache.high >= 127;
	if (i >= IntegerCache.low && i <= IntegerCache.high)
	  return IntegerCache.cache[i + (-IntegerCache.low)];
	return new Integer(i);
}
```
从上面的代码可以看出Integer缓存的结果区间,不错我们可以通过VM参数来修改缓存的High值。

查看源码可以看到Byte缓存了(byte)(-128~127)之间的值,Character缓存了(char)(0-127)之间的值,Short缓存了(short)(-128~127)之间的值,Long缓存了(long)(-128~127)之间的值

### 类型之间的强制转换

关于强制转换，我们同样先看一段代码:

```
int s = 32768;
    
System.out.println((short)s);//-32768

```

结果为什么会这样呢？因为int是4个字节，也就是32位,而short是2个字节也就是16位，其最高位需要用来表示是正数还是负数，所以其最大值只有32767。当int转换为short的时候高位就被截断了.

当一个小数据向大数据转换的时候，系统会自动将其转换而不用我们手动添加转换，这些类型由"小"到"大"分别为 (byte，short，char)--int--long--float—double。这里我们所说的"大"与"小",并不是指占用字节的多少,而是指表示值的范围的大小。

比如以下的代码都是编译通过的

```java
	byte b = 12;
    int i = b;
    long lon = i;
    float f = lon;
    double d = f;

```

如果低级类型为char型，向高级类型（整型）转换时，会转换为对应ASCII码值，例如

```java
char c='c'; int i=c;

System.out.println("output:"+i);//output:99

```

对于byte,short,char三种类型而言,他们是平级的,因此不能相互自动转换,可以使用下述的强制类型转换。

```
short i=99 ; char c=(char)i; 
System.out.println("output:"+c);//output:c
```

**所以，高等级的数据类型转换为低等级的数据类型的时候一定要慎重，尽量不去做这种操作**