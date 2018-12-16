title: fastjson中SerializerFeature解析
date: 2015-11-15 12:49:01
tags: [json，java]
categories: java
keywords: fastjson,SerializerFeature
---

SerializerFeature在fastjson中扮演着一个极其重要的角色，通过它我们可以定制序列化，那么它本身是如何实现的呢？如此的设计有有何出彩之处呢？本篇文章将会重点围绕这些要素进行分析。<!--more-->
### 1. 认识SerializerFeature
```java
public enum SerializerFeature {
    QuoteFieldNames，
    
    UseSingleQuotes，
   
    WriteMapNullValue，
    
    WriteEnumUsingToString，
    
    UseISO8601DateFormat，
    /**
     * @since 1.1
     */
    WriteNullListAsEmpty，
    /**
     * @since 1.1
     */
    WriteNullStringAsEmpty，
    /**
     * @since 1.1
     */
    WriteNullNumberAsZero，
    /**
     * @since 1.1
     */
    WriteNullBooleanAsFalse，
    /**
     * @since 1.1
     */
    SkipTransientField，
    /**
     * @since 1.1
     */
    SortField，
    /**
     * @since 1.1.1
     */
    @Deprecated
    WriteTabAsSpecial，
    /**
     * @since 1.1.2
     */
    PrettyFormat，
    /**
     * @since 1.1.2
     */
    WriteClassName，

    /**
     * @since 1.1.6
     */
    DisableCircularReferenceDetect，

    /**
     * @since 1.1.9
     */
    WriteSlashAsSpecial，

    /**
     * @since 1.1.10
     */
    BrowserCompatible，

    /**
     * @since 1.1.14
     */
    WriteDateUseDateFormat，

    /**
     * @since 1.1.15
     */
    NotWriteRootClassName，

    /**
     * @since 1.1.19
     */
    DisableCheckSpecialChar，

    /**
     * @since 1.1.35
     */
    BeanToArray，
    /**
     * @since 1.1.37
     */
    WriteNonStringKeyAsString，
    /**
     * @since 1.1.42
     */
    NotWriteDefaultValue
    ;

	//枚举的构造方法不能是public
   	private SerializerFeature(){
		 mask = (1 << ordinal());
	}

    private final int mask;

    public final int getMask() {
        return mask;
    }

    public static boolean isEnabled(int features， SerializerFeature feature) {
        return (features & feature.getMask()) != 0;
    }
    
    public static boolean isEnabled(int features， int fieaturesB， SerializerFeature feature) {
        int mask = feature.getMask();
        
        return (features & mask) != 0 || (fieaturesB & mask) != 0;
    }

    public static int config(int features， SerializerFeature feature， boolean state) {
        if (state) {
            features |= feature.getMask();
        } else {
            features &= ~feature.getMask();
        }

        return features;
    }
```

SerializerFeature是一个枚举类型(为什么不使用常量呢?这个会在接下来的文章当中提到)，其中定义了多个枚举(我们可以将其当成类来处理，只是这个类有点特殊而已)，比如我们常用的**BrowserCompatible(游览器兼容，使用它会使得json串中的key加上双引号)**以及**PrettyFormat(格式化)**，且提供了一个默认构造方法:
```java
 private SerializerFeature(){
	mask = (1 << ordinal());
}
```
每实例化一个Enum的时候就会调用一次构造方法，其中的1&lt;&lt;ordinal()是此次讲解的重中之重。


### 2. 认识SerializerFeature的构造方法

```java
private SerializerFeature(){
	mask = (1 << ordinal());
}
```
`ordinal()`方法根据枚举定义了的个数返回不同的数值，如果定义了五个枚举，那么该方法根据顺序依次返回0-4的五个数字。比如**QuoteFieldNames，UseSingleQuotes，WriteMapNullValue依次返回的就是0，1，2这三个数字**，所以根据这样定义枚举相对应的mask值就是1，2，4，8，16，32...(**2的倍数，转换成2进制之后其特殊性就在于只有首位才为1**)	 	
比如4的二进制:100，4096的二进制:1000000000000


### 3. 注册或者移除SerializerFeature

```java
public static int config(int features， SerializerFeature feature， boolean state) {
    if (state) {
        features |= feature.getMask();
    } else {
        features &= ~feature.getMask();
    }
    return features;
}
```
**第一个参数表示默认的SerializerFeature,fastjson在运行过程中加载了默认的SerializerFeature**
```java
static {
    int features = 0;
    features |= SerializerFeature.QuoteFieldNames.getMask();
    features |= SerializerFeature.SkipTransientField.getMask();
    features |= SerializerFeature.WriteEnumUsingToString.getMask();
    features |= SerializerFeature.SortField.getMask();
    // features |=
    // com.alibaba.fastjson.serializer.SerializerFeature.WriteSlashAsSpecial.getMask();
    DEFAULT_GENERATE_FEATURE = features;
}
```
**第二个参数表示需要注册/移除的特性**

**第三个参数表示是注册还是移除特性的状态**

**注册过程:**
```java

 				0000 0010
			|(或)	0000 1000
			|(或)	0010 0000  
----------------------------------------------------------------------------
				0010 1010  (此时表示已经注册了三个特性,有几个1就有几个特性)
```
**移除过程:**
```java
			
			~ 0000 0010 = 1111 1101
				
				0010 1010
		   	&(与)	1111 1101
-----------------------------------------------------------------------
				0010 1000 (此时已经只具有了两个特性,已经被移除了一个)
```
是不是觉得位运算很神奇呢？不得不让人感叹细微之处方见大智慧呀！

### 4. 验证是否存在该特性
```java
public static boolean isEnabled(int features， SerializerFeature feature) {
    return (features & feature.getMask()) != 0;
}
    
public static boolean isEnabled(int features， int fieaturesB， SerializerFeature feature) {
    int mask = feature.getMask();
    
    return (features & mask) != 0 || (fieaturesB & mask) != 0;
}

```
和config使用的方法一致,不在做过多累述。

### 5. 为什么使用枚举

以前我们在代码中,定义使用过的常量,我们可能会这样定义
```java
public class State {
	public static final int ON = 1;
	public static final int OFF= 0;
}

class test {
	public void showState(int state) {
		System.out.println("现在的状态是:"+state);
	}
}
```
既然都用了这么久了，有什么不好的呢？
1. 它不是类型安全的。你必须确保是int
2. 你还要确保它的范围是0 和1
3. 很多时候你打印出来的时候，你只看到1和0 

导致阅读你代码的人并不知道你的意图。所以更多的时候我们应该使用枚举,而不是常量。
