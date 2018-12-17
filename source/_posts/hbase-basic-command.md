---
title: hbase的常用基础命令
date: 2018-12-17 13:25:41
tags:
	- HBase
categories: HBase
---

![hbase](/images/hbase-command/hbase-logo.png)
<!--more-->

1. 查看hbase状态
```
status
```
![status](/images/hbase-command/hbase-status.png)


2. 列出所有表
```
list
```
![list](/images/hbase-command/hbase-list.png)

3. 新建表(需要指定列簇)
```
create 'tableName','columnFamilyName1','columnFamilyName2'
```
![create](/images/hbase-command/hbase-create.png)

4. 追加一个列簇
```
alter 'tableName','columnFamilyName3'
```
![alter](/images/hbase-command/hbase-alter.png)

5. 删除一个列簇
```
alter 'tableName',{NAME=>'columnFamilyName',METHOD=>'delete'}
```
![alter-delete](/images/hbase-command/hbase-alter-delete.png)


6. 查看一个表的描述信息(可以查看有哪些列簇)
```
desc 'tableName'
```
![desc](/images/hbase-command/hbase-desc.png)


7. 插入数据
```
put 'tableName','rowkey','columnFamilyName:columnName','columnValue'
```
![put](/images/hbase-command/hbase-put.png)

8. 获取数据
```
get 'tableName', 'rowkey'

get 'tableName', 'rowkey', 'columnFamilyName:columnName'

```
![get](/images/hbase-command/hbase-get.png)

	get命令还可以指定版本等信息,可以通过 `help 'get' `查看

9. scan的使用
![scan](/images/hbase-command/hbase-scan.png)

	scan命令的更多介绍将在下一篇文章介绍

10. 删除某一列的数据
```
delete 'tableName','rowKey','columnFamilyName:columnName'
```
![delete](/images/hbase-command/hbase-delete.png)

11. 删除某一行的数据
```
deleteall 'tableName','rowKey'
```
![deleteall](/images/hbase-command/hbase-deleteall.png)


12. 删除一张表
```
disable 'tableName'
drop 'tableName'
```
![drop](/images/hbase-command/hbase-drop.png)

13. 查看表是否存在
```
exists 'tableName'
```
![exists](/images/hbase-command/hbase-exists.png)

14. 清空表数据
```
truncate 'tableName'
```
![truncate](/images/hbase-command/hbase-truncate.png)

15. 参考文档
<https://learnhbase.net/>