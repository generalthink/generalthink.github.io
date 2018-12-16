title: Mysql必知必会读书笔记
date: 2016-08-30 22:15:41
tags: mysql
categories: mysql
description: 重温一下MySQL,最近看了MySQL必知必会这本书,对书上一些以前模糊的地方做了一下笔记。
keywords: mysql mysql基本使用 MySQL必知必会
---

**Mysql必知必会笔记(需要有一定Mysql基础)**


--查看有哪些库
```
show databases;
```

--查看当前库中有哪些表
```
show tables
```

--查看表中有哪些列
```sql
show columns from  table_name; 或者   describe table_name;
```

--查看服务器状态
```sql 
    show status;
```

--查看建表语句或者创建数据库的语句
```sql
show create table table_name 和 show create database database_name;
```

--用来显示授权用户的安全权限
```sql 
show grants
```

--用来显示服务器错误或者警告
```sql 
show errors 和 show warnings
```

--limit语句

    ```sql
    #表示从第三行(包括)开始取值，取四条数据(mysql的行是从0开始的)
    select id from table_name limit 3,4;

    #Mysql5版本支持同样语义的写法:
    select id from table_name limit 4 offset 3;
    ```
--sql语句中使用全限定名称
```sql
select table_name.column_name from database_name.table_name;
```

--order by 的说明
order by 多个列的时候,只有当第一个列相同的时候,才会根据第二个列进行排序,以此类推,但是如果第一个列是唯一的话,
那么就不会根据其他列进行排序

--mysql在执行匹配的时候默认不区分大小写
```sql
#这样可能查询出name='li'的结果
select id,name from table_name where name = 'Li';
```

--判断是否为空或者不为空
```sql 
select id from table_name where id is null; 或者 select id from table_name where id is not null;
```

--and的优先级大于or优先级
```sql
#表示查询age < 10的任何数据和(id = 13并且age > 20)的数据
select id,name from table_name where age < 10 or id = 13 and age > 20;
```

--in和or的比较
_ in和or的作用一致,但是in比or更加清晰,而且一般来说in的效率比or高,而且in还可以包含子查询

--使用通配符
1. %表示任意字符出现任意次数,比如select id from table_name where name like 'jack%';表示查询所有名称为jack开头的数据
通配符可以在任意位置并且可以使用多个通配符,值得注意的是%代表搜索模式中给定位置的0个、1个或者多个字符。

2. _通配符只匹配一个字符
3. 不要过度使用通配符,因为它的效率不高,不要讲通配符放在匹配模式的开始处,因为这样搜索起来是最慢的,如果你在某个字段使用了索引,然后使用通配符进行查询,如果将通配符放到开始处,那么就不会使用索引。不要把通配符放错位置了

--Mysql中的正则表达式
1. 使用regexp关键字
```sql 
select id from table_name where name regexp 'zhangsan';
```
2. 使用点(.)来表示任意一个字符
```
#找出id为1000,2000,3000等的name
select name from table_name where id regexp '.000';
```
3. 如果数据库中存在id:10 name:pack10 这样一条数据,那么以下两条sql中第一条不会返回任何数据,而第二条会返回
```
select id from table_name where name like '10';
select id from table_name where name regexp '10';
```
因为like匹配整个列,如果数据在列值中出现的话,那么like不会找到它，除非使用%进行模糊匹配。
但是regexp则可以在列值内进行匹配,如果找到则会返回,当然可以使用^10$这种方式进行匹配整个列值。
4. 匹配是不区分大小写的,除非手动加上binary关键字,比如name regexp binary 'zhangsan';

5. or匹配,比如 id regexp '100|200|300';

6. 匹配单一的字符,那么使用[]将内容括起来,比如我要查看a,b,1,2这几个字符,那么就是name regexp '[ab12]test';

7. 匹配非操作，使用^,比如[^123]将会匹配非1,2,3这些字符的数据

8. 匹配范围,可以使用[0-9A-Za-z]等方式使用
9. 特殊字符的匹配
```
匹配特殊字符,如果想要在正则表达式中匹配. | [ ]这些正则中特殊的字符,那么需要进行转义,在Mysql中使用\\进行转义,比如regexp '\\.',
如果需要匹配\那么就需要使用\\\
```
10. 元字符的引用
```
    \\f    换页
    \\n    换行
    \\r    回车
    \\t    制表
    \\v    纵向制表
```

11. 匹配字符类
```
    [:alnum:]   任意字母和数字,(同[a-zA-Z0-9])
    [:alpha:]   任意字母(同[a-aA-Z])
    [:blank:]   空格和制表,(同[\\t])
    [:cntrl:]   ASCII控制字符(ASCII 0到31和127)
    [:digit:]   任意数字(同[0-9])
    [:graph:]   与[:print:]相同,但不包含空格
    [:lower:]   任意小写字母,(同[a-z])
    [:print:]   任意可打印字符
    [:punct:]   既不在[:alpha:]又不在[:cntrl:]中的任意字符
    [:space:]   包含空格在内的任意空格字符(同[\\f\\n\\r\\t\\v])
    [:upper:]   任意大写字母(同[A-Z])
    [:xdigit:]  任意十六进制数字(同[a-fA-F0-9])
```

12. 重复元字符

    ```
        *       0个或多个匹配
        +       1个或多个匹配(同{1,})
        ?       0个或1个匹配(同{0,1})
        n       指定数目的匹配
        {n,}    不少于指定数目的匹配
        {n,m}   匹配数目的范围,m不超过255
    ```

    例子：
    ```sql
    #得到的答案为(1 stricks)和tiny (5 strick)
    select name from table_name where name regext '\\([1-9] stricks?\\)';

    #匹配连在一起的任意四位数字: 
    select id,name from table_name where id regext '[[:digit:]]{4}';
    ```
       

13. 为了匹配特定位置的文本,需要使用定位符
```
    元字符         说明
    ^              文本的开始
    $              文本的结尾
    [[:<:]]        词的开始
    [[:>:]]        词的结尾

    select '.3abc' REGEXP '^[0-9\\.]'返回结果为1,因为'.3abc'中包含了以数字开头的数据
```
--Mysql函数
1. concat()拼接串,参数需要一个或者多个串,各个串之间用逗号间隔

2. rtirm()删除右边空格,ltrim()删除左边空格,trim删除左右两边空格
3. 常用文本处理函数
```
    Left()              返回串左边的字符
    Length()            返回串的长度
    Locate()            找出串的一个子串
    Lower()             将串转换成为小写
    LTrim()             去掉左边的空格
    Right()             返回串右边的字符
    RTrim()             去掉右边的空格
    SubString()         返回子串的字符
    Upper()             将串转换成大写
```
4. 常用日期和时间处理函数
```
AddDate()           增加一个日期(天、周等)
AddTime()           增加一个时间(时、分等)
CurDate()           返回当前日期
CurTime()           返回当前时间
Date()              返回日期时间的日期部分
DateDiff()          计算两个日期之差
Date_Add()          高度灵活的日期运算函数
Date_Format()       返回一个格式化的日期或者字符串
Day()               返回一个日期的天数部分
DayOfWeek()         对于一个日期,返回对应的星期几
Hour()              返回一个时间的小时部分
Minute()            返回一个时间的分钟部分
Month()             返回一个日期的月份部分
Now()               返回当前日期和时间
Second()            返回一个时间的秒部分
Time()              返回一个日期时间的时间部分
Year()              返回一个日期的年份部分
```

5. 聚集函数
    count(*)对表中行的数目进行计数,不管表列中包含的是空值(NULL)还是非空值
    count(column)对特定列中具有值的行进行计数,忽略NULL值

    Max()一般用于找出最大的数值或者日期值,但是在用于文本数据时,如果数据按照相应的列进行排序,则Max()返回最后一行。
    Max(),Min()均忽略列值为NULL的行

--使用group by
1. group by 可以包含任意列
    如果在group by 子句中嵌套了分组,数据将在最后规定的分组上进行汇总
    group by子句中列出的每个列都必须是检索列或者有效的表达式(但不能是聚集函数)
    除掉聚集计算语句外,select语句中的每个列都必须在group by子句中给出
    如果分组列中具有NULL值,则NULL将作为一个分组返回,如果列中有多行NULL值,它们将分为一组。
    group by子句必须出现在where之后,order by 之前

2. 使用having来进行对分组进行过滤,例如:
```sql
select id,sum(score) from table_name group by id having sum(score) > 120;
```

--having和where的区别

where在数据分组前进行过滤,having在数据分组后进行过滤

--使用子查询
1. 使用子查询的时候,必须保证select语句与where子句中存在相同数目的列
2. ANSI SQL规范首选inner join语法
3. 自连接(将一个表当做两个表看)的使用:
```sql
    select id,name from table_name where id = (select id from table_name where score > 60);
    select id,name from table_name as t1,table_name as t2 where t1.id == t2.pid;
```
4. MySQL中的各种连接
    左外连接也叫左连接（left outer join也可以简写为left join）
      显示左表的所有数据，然后根据条件与右表进行匹配，如果有匹配的就加在左表的后面，如果有多条匹配数据，则显示多条。
    没有的话，该行的右表就以null值填充。

    右连接（right outerjoin 也可以简写为right join）
      显示右表的所有数据，然后根据条件与左表匹配，如果有匹配的就加在左表的后面，如果有多条匹配数据，则显示多条。
    没有的话，该行以null值填充。（和左连接类似）

    何为左表、右表呢 ？在join的左边就称为左表，右边就称为右表

--union的使用
1. 组合查询,利用union可以将多条select语句将它们的结果组合成当个结果集,union规则：
    union必须由两条或者两条以上的select语句组成,语句之间可以用union分割
    union中的每个查询必须包含相同的列、表达式或聚集函数(不过各个列不需要以相同的次序输出)
    列数据类型必须兼容：类型不必完全相同,当必须是DBMS可以隐含的转换的类型(例如:不同的数值类型或不同的日期类型)

2. union会将返回的结果集中重复的数据行自动取消(即只会返回一行),如果想匹配所有返回行,这需要使用union all

--全文本搜索
1. MyIsam引擎支持全文本搜索,InnoDB不支持
2. 启用全文本搜索支持,一般在创建表的时候启用全文搜索
```
    create table tabl_name (
        id int not null auto_increment,
        name varchar(200) not null default '',
        note_text text null,
        fulltext(note_text)
    )engine=myisam
```
3. 在导入的时候不要使用fulltext,因为会维护一份索引导致导入过慢

4. Match()指定被搜索的列,Against()指定要使用的搜索表达式使用:select node_text from table_name where Match(node_text) against('rabbit');

5. 传递给Match()的值必须与FullText()定义的相同,如果指定多个列,则必须列出他们(而且次序正确)
6. fullText不区分大小写,除非使用binary关键字
7. 使用Like也能达到相同的想过,但是使用FullText会对返回的结果集进行排序,出现的关键字在前面的可能会最先返回,即具有高优先级的最先返回(可能这正是你想要的行)
8. 
```sql
#将其放在select中如果文本中包括rabbit返回的rank值就大于0,否则就为0,这也证明了fullText是如何排除行,如何排序结果的
select node_text，Match(node_text) against('rabbit') as rank from table_name;
```
--插入
1. 使用insert的时候最好指定列
2. 如果表的定义允许,则可以在insert操作中省略某些列,省略的列必须满足以下某个条件。
    该列允许为NULL值(无值或空值)
    在表定义中给出默认值，这表示如果不给出值，将使用默认值
3.MySql经常被多个客户访问,对处理什么请求以及用什么次序管理是Mysql的任务,insert操作可能很耗时(特别是有很多索引需要更新的时候),而且他可能降低等待处理的select语句的性能。如果数据检索是重要的(通常是这样),则可以通过在insert和into中间添加关键字LOW_PRIORITY,指示Mysql降低insert语句的的优先级。
4. 插入多条数据
```sql
#此举可以提高insert的性能
insert into table_name(id,name) values(1,'zhangsan'),('2','lisi');
```
56.insert select语法
```sql 
insert into table_name(id,name) select id,name from table_name2
```

--更新和删除
1. 使用update的时候最好指定条件,也可以使用子查询
2. 如果用update语句更新多行,并且在更新的时候其中的一行或者多行出现错误,则整个update操作被取消(错误发生前更新的所有行为被恢复到它们原有的值),如果想即使发生了错误,也继续进行更新,可以使用ignore关键字,如下所示：update ignore table_name

3. delete删除的是表的行,而不包含表本身
4. 如果想要快速删除表中的所有数据,可以使用truncate table语句,它完成相同的工作,但是它更快(truncate实际上是删除原来的表并重新新建一个表,而不是逐行删除表中的数据)

--创建和操纵表
1. 如果你仅仅现在一个表不存在的时候创建它,那么可以使用create table table_name if not exists 语句
2. mysql不允许使用函数作为列的默认值,它只支持常量
3. 常用引擎
    InnoDB是一个可靠的事务处理引擎，它不支持全文搜索
    MyIsam是一个性能极高的引擎,它支持全文搜索
    Memory在功能上等同于MyIsam,但是由于数据存储在内存中,速度很快(特别适合于临时表)
4. 引擎类型可以互用,但是外键不能跨引擎
5. alter table用于更改表结构,必须给出以下信息:
    在alter table之后必须要给出要更改的表名(该表必须存在,否则将出错)
    所做更改的列表
6. 删除表:```drop table table_name;```
7. 重命名表:```rename table table_name1 to table_name2;```
    重命名多个表:```rename table table1 to table2,table3 to table4;```

--视图的使用
Mysql5之后才有视图
