title: infobright使用以及问题汇总
date: 2016-06-09 12:49:40
tags: [infobright,mysql]
categories: mysql
keywords: java infobright myisam mysql brighthouse
---

最近好久没有写博客了，主要是最近太忙，一直都在加班，所以没有时间写博客，但是这个博客还是耗费了自己比较大的心血的，从备案到组建，一路过来一路心酸呀，工作一年以来自己进步也蛮大，处理事情的能力也得到了一定的提高，之前接触到的工作使用比较多的是Infobright，由于工作内容涉及到了大量的查询，也不使用事务，所用我们的数据库使用的最多的就是MyIsam以及Brighthouse引擎，但是Infobright的资料比较少，现在就针对自己在这过程中遇到的问题进行一个总结。

<!--more-->

### 关于Infobright

Infobright是基于专利技术的列式数据库，一个基于MySQL开发的开源数据仓库（Data Warehouse）软件，可作为MySQL的一个存储引擎来使用，SELECT查询与普通MySQL无区别。

### Infobright的优点

1. 高压缩比率，平均压缩比可达10：1。（经测试，我们的1500万条697M日志数据压缩比例为6：1，压缩后数据大小仅为114M）

2. 列存储，即使数据量十分巨大，查询速度也很快。（经测试，一条在infobright中的复合查询要30秒，在mysql中要一分多）

3. 不需要建索引，就避免了维护索引及索引随着数据膨胀的问题。把每列数据分块压缩存放，每块有知识网格节点记录块内的统计信息，代替索引，加速搜索。

4. 单一台服务器可以高效地读写30T数据。具有可扩展性，这里是指对于同样的查询，当数据量是10T时，它耗费的时间不应该比1T数据量时慢太多，基本是一个数量级内。

**注:实际上infobright取一条记录比mysql要慢很多，但它取100W条记录会比mysql快。所以比较适用于日志、或汇总的大量的数据。**

### Infobright的限制

1. 不支持数据更新：社区版Infobright只能使用“LOAD DATA INFILE”的方式导入数据，不支持INSERT、UPDATE、DELETE

2. 不支持高并发：只能支持10-18多个并发查询

### Infobright的安装

#### 下载地址

[infobright下载地址](http://www.infobright.org/index.php/download/ICE/)


#### Linux下的安装
网上很多Linux下的安装版本，但是大都是抄袭张宴的博客，如果需要看Linux下安装，去看张宴的博客就行了

#### Windows下的安装

**安装步骤**
![Infobright安装](/images/hexo-infobright_setup_1.png)

![Infobright安装](/images/hexo-infobright_setup_2.png)

![Infobright安装](/images/hexo-infobright_setup_3.png)

**查看安装是否成功**
![Infobright服务启动](/images/hexo-infobright_setup_4.png)
如果服务没有启动，可以通过命令手动启动服务

```
net start infobright  --启动服务
net stop infobright  --暂停服务
```

![Infobright连接](/images/hexo-infobright_setup_5.png)

![Infobright连接](/images/hexo-infobright_setup_6.png)

**Infobright的配置以及使用**
![Infobright的配置](/images/hexo-infobright_use_2.png)
![创建BrightHouse数据库表](/images/hexo-infobright_use_1.png)

### Infobright在Java中的使用
安装好了之后，我们应该如何使用呢？其实使用方式和Mysql的正常使用方式只有细微的区别，ICE版本不支持INSERT,UPDATE,DELETE语句,收费版本的可以,Select语句和Mysql使用一致。

1. **建表语句**

```sql
CREATE TABLE `user_info` (
  `id` bigint(20) NOT NULL,
  `name` varchar(20) NOT NULL,
  `age` int(3) NOT NULL,
  `user_desc` varchar(1024) NOT NULL default ''
) ENGINE=BRIGHTHOUSE DEFAULT CHARSET=utf8;
```

2. **在代码中的使用**

查询语法和Mysql语法一致，没有任何区别(不支持insert、update、delete)


3. **注意事项**
Infobright引擎与之前MyISAM引擎不同的一些特性(会影响到我们的特性):

    - 不支持数据更新，只能使用“LOAD DATA INFILE”的方式导入数据，不支持INSERT、UPDATE、DELETE；如果有修改表的操作，不能使用Infobright的表。

    - 建表时不能有AUTO_INCREMENT自增、unsigned无符号、unique唯一、主键PRIMARY KEY、索引KEY。

    - varchar、char字段和TEXT字段的长度可能会比之前的短。例如使用text字段，最长支持65535字节，目前只能支持22000左右。

	- varchar和char字段还可以使用comment lookup，comment lookup能够显著地提高压缩比率和查询性能。但是使用lookup不能离散度太高，例如一个表不能离散超过20000条。
	comment lookup使用很简单，在创建数据库表的时候如下定义即可：
	
	```sql
	CREATE TABLE `user_info2` (

	  `id` int(20)  NOT NULL,
	  `name` varchar(20) NOT NULL COMMENT 'look up',
	  `age` int(3) NOT NULL,
	  `user_desc` varchar(1024) NOT NULL default ''
	) ENGINE=BRIGHTHOUSE DEFAULT CHARSET=utf8;

	```
	
