---
title: MongoDB初学者最常用的10个命令
date: 2019-09-26 14:23:03
tags: mongodb
categories: mongodb
---

#### 1. 登录mongodb

以下命令可以用于登录mongodb数据库，但是需要保证用户你声明的数据库中存在对应的用户和密码
```
mongo --host <hostName> --port <port> -u <username> -p <password> --authenticationDatabase <dbname>

mongo --host 192.168.140.11 -u test -p 123456 --authenticationDatabase test_db

```
<!--more-->

#### 2. 列出所有的数据库

  当你以适当角色的用户身份登录后，可以使用以下命令查看所有数据库

```
show dbs
```

#### 3. 选择要使用的数据库

要开始使用特定的数据库，可以用以下命令

```
use <databaseName>
```

#### 4. 创建用户

  当你想让不同的用户拥有不同的权限的时候可以使用以下命令

```
use <databaseName>
db.createUser({ user: '<username>', pwd: '<password>', roles: [ { role: "readWrite", db: "<databaseName>" } ] });

例子:
use admin
db.createUser({ user: 'admin', pwd: '123456', roles: [ { role: "userAdminAnyDatabase", db: "admin" } ] });

```

#### 5. 列出所有的集合，用户以及角色

```
// 列出当前database下所有的集合:
show collections;
db.getCollectionNames();

// 列出当前database下所有的用户
show users;
db.getUsers();

// 列出当前dababase下所有角色
show roles;
```

不同的角色对应的权限，最直接之处就在于没有权限有些命令就无法执行

#### 6. 创建集合

  下面的命令用户创建集合，更详细命令可以查看[官方文档](https://docs.mongodb.com/manual/reference/method/db.createCollection/)

```
db.createCollection("collectionName");
```

#### 7. 将文档插入到集合中

  集合一旦创建之后，下一步就是创建一个或多个文档插入到集合中

```
// 插入单个文档
db.<collectionName>.insert({field1: "value", field2: "value"})

// 插入多个文档
db.<collectionName>.insert([{field1: "value1"}, {field1: "value2"}])
db.<collectionName>.insertMany([{field1: "value1"}, {field1: "value2"}])


```

#### 8. 保存或者更新文档

  保存命令可用于更新现有文档或根据传递给它的文档参数插入新文档。如果传递的_id与现有文档匹配，则文档将更新。否则，将创建一个新文档。在内部，保存方法使用插入或更新命令。

```
db.<collectionName>.save({"_id": new ObjectId("123456"), field1: "value", field2: "value"});
```

#### 9. 显示集合记录

```
// 获取所有记录
db.<collectionName>.find();

// 获取指定数量的记录
db.<collectionName>.find().limit(10);

// 根据id获取记录
db.<collectionName>.find({"_id": yourId})

// 返回记录中特定field的值
// 类似返回select field1,field2 from table
db.<collectionName>.find({"_id": ObjectId("someid")}, {field1: 1, field2: 1});
// 不返回field1的数据
db.<collectionName>.find({"_id": ObjectId("someid")}, {field1: 0});

// 文档记录数
db.<collectionName>.count();
```

#### 10. 管理命令
以下是一些管理命令，这些命令可能有助于查找集合详细信息，例如存储大小，总大小和总体统计信息

```
// 获取集合的统计信息,比如空间占用，总大小，引擎信息等
db.<collectionName>.stats()
db.printCollectionStats()

//获取集合的延迟统计信息,比如读写的次数,时间等等
db.<collectionName>.latencyStats()

// 获取数据和索引的集合大小
// 集合的大小
db.<collectionName>.dataSize()
// 集合中存储文档的总大小
db.<collectionName>.storageSize()
// 集合数据和索引的总大小(以字节为单位)
db.<collectionName>.totalSize()
// 集合中所有索引的总大小
db.<collectionName>.totalIndexSize()
```