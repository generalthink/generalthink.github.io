---
title: HBase scan命令详解
date: 2018-12-17 15:59:23
tags:
	- HBase
categories:
	- HBase
---

hbase中scan命令是我们经常使用到的，而filter的作用尤其强大。这里简要的介绍下scan下filter命令的使用.

### 插入scan命令需要的数据
这里模拟了部分微博评论的数据,然后使用代码插入数据到hbase,代码就不列出来了比较简单。
```java
public class Comment {
    //1-->普通文章，2--->热点文章
    Integer articleType;
    //文章id
    String articleId;
    String userId;
    long timestamp;
    //comment content,暂时只考虑文本
    String commentContent;
}
```
<!--more-->
hbase的表名称为zy_comment,列簇info下有articleType以及commentInfo两个列。commentInfo的value为上面Comment类的json字符串,插入的数据如下所示
![插入数据](/images/hbase-command/hbase-scan-readydata1.png)
![插入数据](/images/hbase-command/hbase-scan-readydata2.png)

### HBase数据顺序
HBase是三维有序存储的，是指rowkey（行键），column key（column family和qualifier）和TimeStamp（时间戳）这个三个维度是依照ASCII码表排序的。（比如A排在a前面）
1. 先rowkey升序排序，
2. rowkey相同则column key升序排序
3. rowkey、column key相同则timestamp降序排序

### 支持的Filter
scan命令我们经常会大量使用Filter,hbase shell提供的filter都可以在hbase client包中找到对应的类，它们都是Filter的子类，很多命令都是通过filter来进行实现的。
![filter](/images/hbase-command/hbase-scan-javafilter.png)

使用`show_filters`命令查看shell中定义了哪些filter常量,如果想要使用shell中未定义的常量,在使用的时候必须手动import filter的全路径。
![filter](/images/hbase-command/hbase-show-filter.png)

### scan的用法

使用 `help 'scan'` 命令可以查看scan的语法以及用法,关于scan命令中filter的使用规则如下:
```
scan 'tableName',{FILTER=>"FilterName(param1,param2,...,paramN)"}
```
{}中的语法是ruby的map的语法，FILTER必须大写,filter的参数是根据构造方法来的，也就是相当于java中的new Filter('param1','param2')等，这里只是省略了new参数而已。
当然同样可以使用ruby中new对象的方式，只是那样就必须使用全限定名称。后面会举一个全限定名称的例子。

在使用Filter的过程中部分filter会用到比较器(CompareOperator.java)以及运算比较符(ByteArrayComparable.java)

```java
@Public
public enum CompareOperator {
    LESS,   // <
    LESS_OR_EQUAL,    // <=
    EQUAL,    // =
    NOT_EQUAL, // <>
    GREATER_OR_EQUAL, // >=
    GREATER, // >
    NO_OP; // 没有任何操作

    private CompareOperator() {
    }
}

```

比较器主要有以下几种:

```
BinaryComparator 按字节索引顺序比较指定字节数组，采用Bytes.compareTo(byte[]),比如:binary:\x00\x00\x02
BinaryPrefixComparator 跟前面相同，只是比较左端的数据是否相同,比如:binaryprefix:\x00\x00
NullComparator 判断给定的是否为空,不常用
BitComparator 按位比较 a BitwiseOp class 做异或，与，并操作，不常用
RegexStringComparator 提供一个正则的比较器，仅支持 EQUAL 和非EQUAL,比如:regexstring:ab*add
SubstringComparator 判断提供的子串是否出现在table的value中。比如：substring:great
```

### scan例子
查询方式通过rowKey来进行查询是最快的，所以rowkey的设计一定要合理，如果不合理会很影响查询速率。但是有时候确实没有办法完全通过rowkey来查询，所以就要借助scan.
scan命令支持的修饰词除了列（`COLUMNS`）修饰词外，HBase还支持`Limit`（限制查询结果行数），`STARTROW`（ROWKEY起始行。会先根据这个key定位到region，再向后扫描）、`STOPROW`(结束行)、`TIMERANGE`（限定时间戳范围）、`VERSIONS`（版本数）、和`FILTER`（按条件过滤行）等

1. 查询user是zhangsan的用户的评论数据,最多只返回10条
` scan 'zy_comment',{LIMIT=>10,FILTER=>"PrefixFilter('zhangsan')"} `
	![scan命令](/images/hbase-command/hbase-scan-prefixfilter.png)


2. 通过startrow,stoprow来进行查询(这种也比较快,实际操作中如果不能通过rowkey)
` scan 'zy_comment',{STARTROW=>'b',STOPROW='e',LIMIT=>10} `
	![scan命令](/images/hbase-command/hbase-scan-startendrow.png)

3. 查询评论内容为666的评论有哪些
` scan 'zy_comment',{LIMIT=>10,FILTER=>"ValueFilter(=,'substring:666')"} `
	![scan命令](/images/hbase-command/hbase-scan-valuefilter.png)

4. 查询热点文章的数据有哪些
```
scan 'zy_comment',{LIMIT=>10,FILTER="SingleColumnValueFilter('info','articleType',=,'binary: \x00\x00\x00\x00\x00\x00\x00\x02')"}
```
	![scan命令](/images/hbase-command/hbase-scan-singlecolumn.png)

	这里binary中的数据一定要是二进制字符串而不是具体的值


5. 查看评论包含great的热点文章评论有哪些
```
 scan 'zy_comment',{LIMIT=>3,FILTER=>"(SingleColumnValueFilter('info','articleType',=,'binary:\x00\x00\x00\x00\x00\x00\x00\x02')) AND (ColumnPrefixFilter('commentInfo') AND ValueFilter(=,'substring:great'))"}
```
	![scan命令](/images/hbase-command/hbase-scan-multifilter.png)

6. 查询评论的前两条
` scan 'zy_comment',{FILTER=>"PageFilter(2)"} `和LIMIT有异曲同工之妙
	![scan命令](/images/hbase-command/hbase-scan-pagefilter.png)

7. 查询rowkey中包含特定前缀的数据
` scan 'zy_comment',{FILTER=>"RowFilter(=,'substring:zhangsan')"} `

	![scan命令](/images/hbase-command/hbase-scan-rowfilter1.png)

	![scan命令](/images/hbase-command/hbase-scan-rowfilter2.png)

8. 使用全限定名称查询articleId是123456的数据
```
import org.apache.hadoop.hbase.filter.SingleColumnValueFilter
import org.apache.hadoop.hbase.filter.CompareFilter
import org.apache.hadoop.hbase.filter.SubstringComparator
scan 'zy_comment', {COLUMNS=>['info:commentInfo'], FILTER => SingleColumnValueFilter.new(Bytes.toBytes('info'), Bytes.toBytes('commentInfo'), CompareFilter::CompareOp.valueOf('EQUAL'), SubstringComparator.new('123456'))}
```

	![scan命令](/images/hbase-command/hbase-scan-filter-fullpath.png)

#### 注意
+ 如果scan中指定了COLUMNS，则FILTER中所使用的列需要包含在所指定的COLUMNS中，否则，filter不起作用。
+ HBase中主要的操作对象是一个个的cell，每个cell都可以有多个版本。如果使用过滤器ValueFilter，就会只有那些符合条件的cell被查出来。跟关系数据库的查询不同，关系数据库查出来的结果中各行都有相同的列。而HBase，查出来的结果中，不同的行会有不同的列。
+ filter不会降低服务方的IO，它会把符合条件的子集传给客户端。即，它是在对查出的结果进行过滤，而不是象原来sql中的where子句。所以，如果要查出的结果中不包含filter需要的列，则filter就不能发挥作用。

### 参考文章

<https://acadgild.com/blog/different-types-of-filters-in-hbase-shell>
<https://blog.csdn.net/u012185296/article/details/47338549>