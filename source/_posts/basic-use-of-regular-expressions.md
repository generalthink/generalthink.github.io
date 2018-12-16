title: 正则表达式基本使用
date: 2018-03-25 12:17:39
tags: [java,正则表达式]
categories: [正则表达式]
description: 关于正则表达式的基本使用
keywords: 正则表达式 环视 零宽表达式
---
正则表达式是一个很强大的模式语言，使用它我们能够解决很多很棘手的问题，有时候使用字符串查找来解决这类问题不是很方便，所以这个时候正则表达式就能帮我们很大的忙。

完整的正则表达式由两种字符构成。特殊字符(specialcharacters,比如*)
称为“元字符”(metacharacters),其他为“文字”(literal),或者是普通文本字符(normaltext characters).

### 如何理解正则表达式

正则表达式是一门语言，同样有着它的语言模式,所以我们要以它的模式来理解它，比如^cat(^表示行开头)的意思是匹配以c字符作为第一行的第一个字符,紧接一个a，紧接一个t的文本.

### 正则表达式的基本使用

#### 普通字符
普通字符包括没有显式指定为元字符的所有可打印和不可打印字符。这包括所有大写和小写字母、所有数字、所有标点符号和一些其他符号。

#### 特殊字符

```
元字符		 名 称 			匹配对象
.       	点号 			匹配除了换行符(\n)以外的任意一个字符
[…] 		字符组 			列出的任意字符
[^…] 		排除型字符组 	未列出的任意字符
^			脱字符 			行的起始位置
$ 			美元符 			行的结束位置
| 			竖线 			匹配分隔两边的任意一个表达式
(…) 		括号 			限制竖线的作用范围，其他功能下文讨论
?			问号			容许匹配一次,当并非必要
+           加号			表示之前紧邻的元素至少需要匹配一次,至多可能任意多次
*           星号           	表示之前紧邻的元素尽可能匹配多次,也可能不匹配
\char 		转义字符		若char是元字符，或转义序列无特殊含义时，匹配
char对应的普通字符,如果是普通字符则反斜线被忽略
{n,m} 		区间量词		表示至少匹配n次,最多匹配m次
\1,\2		反向引用		匹配之前的第一、第二组括号内的字表达式匹配的
文本

```
如果要匹配元字符，都需要加上反斜杠('\')转义,在字符组内部无效

#### 量词

```
忽略优先量词: *?,+?,??,{n}?,{n,}?,{n,m}?   量词在正常情况下都是“匹配优先”的，匹配尽可能多的内容。相反，这些忽略
优先的量词会匹配尽可能少的内容，只需要满足下限，匹配就能成功

匹配优先量词： *、+、?、{num,num}

占有优先量词: ?+  *+  ++  {m,n}+    这些量词目前只有java.util.regex 和 PCRE (以及PHP)提供,占有优先量词类似普通的匹配优先量词，不过他们一旦匹配某些内容，就 不 会 “交还”。

```

#### 其他通用规则

```
\xXX	编号在 0 ~ 255 范围的字符，比如：空格可以使用 "\x20" 表示
\uXXXX	任何字符可以使用 "\u" 再加上其编号的4位十六进制数表示，比如："\u4E2D"
\t 		制表符
\n 		换行符
\r 		回车符
\s 		任 何 “空白”字 符 （例如空格符、制表符、进纸符等）
\S 		除\s 之外的任何字符
\w 		[a-zA-ZO-9](\w+中很有用，可以用来匹配一个单词)
\W 		除\w之外的任何字符，也就是[Aa-zA-ZO-9]
\d 		[0-9],即数字
\D 		除\d以外的任何字符，即[^a-zA-Z0-9]
\B		匹配非单词边界，即左右两边都是 "\w" 范围或者左右两边都不是 "\w" 范围时的字符缝隙
\b  	匹配单词边界
```

### 字符组
字符组的意思表示匹配若干个字符之一,比如gr[ae]y,表示匹配的结果是grey或者是gray.请注意，在字符组以外，普通字符（例如gr[ae]y 中的g和r)都有“接下来是（and then)”
的意思——“首先匹配g，接下来是r”。这与字符组内部的情况是完全相反的。字
符组的内容是在同一个位置能够匹配的若干字符，所以它的意思是“或”。

在字符组内部，字符组元字符‘-’表示范围的意思,比如[123456]表示1到6之间的任意一个数字，与[1-6]是一样的,同理[0-9A-Za-z]表示匹配9个数字与52个字母中的其中一个，顺序无所谓。我们还可以随心所欲地把字符范围与普通文本结合起来:[0-9A-Z_!.?]能够匹配一个数字、大写字母、下画线、惊叹号、点号，或者是问号。

请注意，只有在字符组内部，连字符才是元字符一否则它就只能匹配普通的连字符号。其
实，即使在字符组内部，它也不一定就是元字符。如果连字符出现在字符组的开头，它表
示的就只是一个普通字符，而不是一个范围。同样的道理，问号和点号通常被当作元字符
处理，但在字符组中则不是如此（说明白一点就是，[0-9A-Z_!] 里面，真正的特殊字符
就只有那两个连字符)。

字符组中的另外一个元字符是^,表示排除的意思，比如[^1-6],表示匹配除了1到6之外的任意字符。而且必须是紧接在字符组的第一个方括号之后才是排除的意思。需要记住的是排除型字符组表示“匹配一个未列出的字符(match a character that’s not listed)”,而不是 “不要匹配列出的字符(don’t match what is listed)”

### 分组和反向引用
小括号 () 可以达到对正则表达式进行分组的效果，分组后会在正则表达式中创建反向引用，反向引用会保存分组的字符片段，这使得我们可以使用这个字符片段。

对于这样一个正则表达式"((\w)\d(test)))"总共存在3组(每个括号是1组,从左到右开始数,整个表达式是第一组,因为它被括号包围了,\w是第二组,test是第三组内容),每个组在程序中都可以获取，正则表达式引擎在匹配的时候会保存整个组的内容，用于以后获取，如果将正则表达式修改为 “((?:\w)\d(test)))”,这样就不能获取\w所捕获的那一组的内容。

在以正则表达式替换字符串的语法中，是通过 $ 来引用分组的反向引用，$0 是匹配完整模式的字符串（注意在 JavaScript 中是用 $& 表示）；$1 是第一个分组的反向引用；$2 是第二个分组的反向引用，以此类推。

在匹配的过程中也是可以使用分组的结果的，表达式后边的部分，可以引用前面 "括号内的子匹配已经匹配到的字符串"。引用方法是 "\" 加上一个数字。"\1" 引用第1对括号内匹配到的字符串，"\2" 引用第2对括号内匹配到的字符串……以此类推，如果一对括号内包含另一对括号，则外层的括号先排序号。换句话说，哪一对的左括号 "(" 在前，那这一对就先排序号。


### 非捕获型括号
括号 "( )" 内的子表达式，如果希望匹配结果不进行记录供以后使用，可以使用 "(?:xxxxx)" 格式,此种方式可以提高效率。
 举例1：表达式 "(?:(\w)\1)+" 匹配 "a bbccdd efg" 时，结果是 "bbccdd"。括号 "(?:)" 范围的匹配结果不进行记录，因此 "(\w)" 使用 "\1" 来引用。

### 环视(或许也被叫做零宽断言)
环视(Perl中叫这个名字)更多的表示的是一个位置,在java中适用,js中也适用,也叫做零宽表达式.

在说环视之前需要注意的一点是,在检查子表达式的过程中，它们本身不会"占用"任何文本，只匹配标记的位置

肯定顺序环视	(?=......)		子表达式能够匹配右侧文本
    原始串：abcdef   正则:abc(?=def)  能够匹配其中的abc,这里?=def匹配的是一个位置,即d的位置,然后这个位置前面有abc三个字符,所以匹配成功

否定顺序序环视	(?!......)		子表达式不能匹配右侧文本
    原始串：abcdef   正则:abc(?!def)  不能匹配其中的abc  但是可以匹配abcddef中的abc

肯定逆序环视	(?<=......)		子表达式能够匹配左侧文本
    原始串：abcdef   正则:(?<=abc)def  能够匹配其中的def,

否定逆序环视	(?<!......)		子表达式不能匹配左侧文本
    原始串：abcdef   正则:(?<!abc)def  不能够匹配def  但是可以匹配 abdef中的def

顺序环视结构中可以使用任意正则表达式，但是逆序环视中的子表达式只能匹配长度
有限的文本。也就是说 ?可以出现在逆序环视中，但 * 和 +则不行


### 正则表达式中的匹配模式
可以在正则的开头指定模式修饰符。

	1. (?i) 使正则忽略大小写。

	2. (?s) 表示单行模式（"single line mode"）使正则的 . 匹配所有字符，包括换行符。

	3. (?m) 表示多行模式（"multi-line mode"），使正则的 ^ 和 $ 匹配字符串中每行的开始和结束。

### 在Java中使用正则表达式
我们需要先了解Java中的两个类

```java
java.util.regex.Pattern
java.util.regex.Matcher
```

简称这两个为“ pattern ”和 “ matcher ”，许多时候我们只会用到这两
个类。简单地说， Pattern对象就是编译好的正则表达式，可以应用于任意多个字符串，
Matcher对象则对应单独的实例，表示将正则表达式应用到某个具体的目标字符串上.

简单应用:

```java
public class RegexTest {
    public static void main(String[] args) {
        String myTest = "this is my 1st test string";
        String myRegex = "\\d+\\w+";
        Pattern pattern = Pattern.compile(myRegex);
        Matcher matcher = pattern.matcher(myTest);
        
        if(matcher.find()) {
            String matchText = matcher.group();
            int matchFrom = matcher.start();
            int matchEnd  =matcher.end();
            System.out.println("matched [ " + matchText + " ] from " + matchFrom + " to " + matchEnd);
        } else {
            System.out.println("don't match");
        }
    }
}

matched [ 1st ] from 11 to 14
```
#### Matcher对象的常用API
通过Matcher对象我们可以修改几个常用的对象：
1. Pattern(usePattern方法)
2. 目标字符串(reset(text)方法)
3. 目标字符串的检索范围(region),默认是整个字符串,但是可以通过region方法修改为目标字符串的某一段,这样某些匹配操作就只能在某个区域进行了。
4. 当前 pattern 的捕获型括号的数目可以通过groupCount查询

当Matcher的正则表达式应用到文本的时候,下面这些方法会比较常用

1. boolean find()
	此方法在目标字符串的当前检索范围中应用 Matcher 的正则表达式，返回的
Boolean 值表示是否能找到匹配。如果多次调用，则每次都在上次的匹配位置之后尝试
新的匹配。没有给定参数的 find 只使用当前的检索 

2. boolean find (int  offset)
	如果指定了整 型参 数 ，匹配尝试会从距离目标字符串开头 offset个字符的位置 开始,这种形式的find不会受当前检索范围的影响,而会把它设置为整个“目标字符串”(它会在内部调用reset方法)
3. boolean matches()
	此方法返回的Boolean值表示matcher的正则表达式能否完全匹配目标字符串中当前检
索范围的那段文本.也就是说,如果匹配成功,匹配的文本必须从检索范围的开头开
始 ，到检索范围的结尾结束(默认情况就是整个目标字符串)
4. boolean lookingAt()
	此方法返回的Boolean值表示Matches的正则表达式能否在当前目标字符串的当前检
索范围中找到匹配.它类似于matches方法,但不要求检索范围中的整段文本都能匹配.

5. String group()
	返回前一次应用正则表达式的的匹配文本

6. Stirng group(int num) 
	返回编号为num的捕获型括号匹配的内容，如果对应的捕获型括号没有参与匹配，则
返回 null。如果num为0，表示返回整个匹配的内容，group(O)就等于group()

### 几个例子

#### 获取返回的数据

```java
public class RegexTest {
    public static void main(String[] args) {
        String myTest = "http://localhost:8080/spring/swagger-ui.html";
        //指定捕获组名称为port
        String myRegex = "http://(\\w+)(?<port>:\\d+)";
        Pattern pattern = Pattern.compile(myRegex);
        
        Matcher matcher = pattern.matcher(myTest);
        int groupCount = matcher.groupCount();
        System.out.println("groupCount = " + groupCount);
        if(matcher.find()) {
        	//返回正则表达式的匹配文本
            System.out.println(matcher.group(0));
            System.out.println(matcher.group(1));
            //group(2)
            System.out.println(matcher.group("port"));
        }
    }
}

输出如下：
groupCount = 2
http://localhost:8080
localhost
:8080
```

#### 非捕获组获取数据

```java
public class RegexTest {
    public static void main(String[] args) {
        String myTest = "http://localhost:8080/spring/swagger-ui.html";
        //注意这里的?:
        String myRegex = "http://(\\w+)(?::\\d+)";
        Pattern pattern = Pattern.compile(myRegex);
        
        Matcher matcher = pattern.matcher(myTest);
        int groupCount = matcher.groupCount();
        System.out.println("groupCount = " + groupCount);
        if(matcher.find()) {
            System.out.println(matcher.group(0));
            System.out.println(matcher.group(1));
        }
    }
}

输出如下：
groupCount = 1
http://localhost:8080
localhost
```

#### 分组引用的使用

```java
 String str = "hello world,hello java";

 //这里的$1是对分组的引用,如果改成(?:hello),下面的代码会报错
 System.out.println(str.replaceAll("(hello)", "$1 my"));

 输出: hello my world,hello my java
```

#### 数据重置

```java
public class RegexTest {
    public static void main(String[] args) {
        String myTest = "hello java";
        String myRegex = "\\w+";
        Pattern pattern = Pattern.compile(myRegex);
        
        Matcher matcher = pattern.matcher(myTest);
       
        //两行数据
        matcher.reset("hello 1998\r\n hello 2018");
        matcher.usePattern(Pattern.compile("\\d+"));
        
        while(matcher.find()) {
            System.out.println(matcher.group());
        }
    }
}
输出如下:
1998
2018
```

#### 匹配非中文数据

```java
public class RegexTest {
    public static void main(String[] args) {
        String myTest = "我是中国人 I'm chinese";
        //去掉^匹配到的将是中文
        String myRegex = "[^\\u4e00-\\u9fa5]+";
        Pattern pattern = Pattern.compile(myRegex);
        Matcher matcher = pattern.matcher(myTest);
        while (matcher.find()) {
            System.out.println(matcher.group());
        }
    }
}
输出如下:
	I'm chines
```

#### 去除连字符

```java
String myTest = "我要要学学学Jaaaaaava";
String myRegex = "(.)\\1+";
System.out.println(myTest.replaceAll(myRegex, "$1"));

输出如下:
	我要学Java
```