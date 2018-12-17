---
title: HBase备份容灾常用命令
date: 2018-12-17 14:32:50
tags:
	- HBase
categories:
	- HBase
---

灾难恢复是个令人神经紧张的话题,但必须面对.HBase虽然是一个分布式的数据库，但是有时候容灾以及数据备份仍然是需要考虑的，而掌握常用的命令正是写这篇文章的意义所在。
本文主要通过案例来讲解CopyTable,Import,Export,Snapshot,希望大家对它们的使用有一个直观的认识。

<!--more-->



### CopyTable
	- 支持时间区间,row区间，改变表名称，改变列族名称，指定是否copy已经被删除的数据等功能
	- CopyTable工具采用scan查询，写入新表时采用put和delete API,全是基于hbase的client api进行读写

1.  首先需要新建好备份表，保证columnFamily一致
![新建备份表](/images/hbase-command/hbase-copytable-ready.png)

2. 在另外一个窗口中,进入hbase/bin目录下，执行以下命令(fileTableNew是备份的表,fileTable是原始表)
```
hbase org.apache.hadoop.hbase.mapreduce.CopyTable --new.name=fileTableNew fileTable
```
![查看结果](/images/hbase-command/hbase-copytable-result.png)


### Export/Import
	1. Export可导出数据到目标集群,然后可在目标集群Import导入数据，Export支持指定开始时间和结束时间，因为可以做增量备份
	2. Export导出工具与CopyTable一样是依赖hbase的scan读取数据

#### Export语法
```
bin/hbase org.apache.hadoop.hbase.mapreduce.Export <tablename> hdfs://namenode:9000/table_bak <version> <startTime> <endTime>
```
#### Import语法

```
bin/hbase -Dhbase.import.version=0.94 org.apache.hadoop.hbase.mapreduce.Import <tablename> <inputdir>
```

1. 查看hbase数据库，只存在fileTable表
![数据准备](/images/hbase-command/hbase-export-ready.png)

2. 执行导出语句
```
#这里存储的路径是存储在hdfs上面的
./hbase org.apache.hadoop.hbase.mapreduce.Export fileTable /usr/local/hbase/fileTable.db

```
![查看导出的表](/images/hbase-command/hbase-export-table.png)

3. 新建需要导入的表，确保导入之前的表和导入后的表结构一致(相同的列簇)
```
create 'fileTableNew','fileInfo','saveInfo'
```

4. 执行导入语句
```
./hbase org.apache.hadoop.hbase.mapreduce.Import fileTableNew /usr/local/hbase/fileTable.db
```

![查看导入的数据](/images/hbase-command/hbase-import-table.png)


### 快照的处理

#### 创建快照
```
snapshot 'myTable','myTableSnapshot-181210'
```

#### 克隆快照
```
clone_snapshot 'myTableSnapshot-181210', 'myNewTestTable'
```

#### 列出快照
```
list_snapshots
```

#### 删除快照
```
delete_snapshot 'myTableSnapshot-181210'
```

#### 恢复数据
```
disable 'myTable'
restore_snapshot 'myTableSnapshot-181210'
```

![创建快照](/images/hbase-command/hbase-snapshot-create.png)

![恢复快照](/images/hbase-command/hbase-snapshot-restore.png)


### 查看Hadoop集群信息

1. `http://ip:50070`
http://ip:50070/jmx可以看到json格式的消息，也可以通过编码获取值，http://ip:50070/jmx?qry=<json中name的值>，比如http://192.168.239.134:50070/jmx?qry=java.lang:type=MemoryPool,name=Survivor%20Space

### 查看hbase集群信息
1. `http://ip:16010`
http://ip:16010/jmx，同样也可以通过qry进行过滤
