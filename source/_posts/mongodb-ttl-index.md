---
title: MongoDB中的定时索引
date: 2019-07-15 15:31:23
tags: MongoDB
---


MongoDB中存在一种索引,叫做TTL索引(time-to-live index,具有生命周期的索引)，这种索引允许为每一个文档设置一个超时时间。一个文档达到预设置的老化程度后就会被删除。
数据到期对于某些类型的信息非常有用，例如机器生成的事件数据，日志和会话信息，这些信息只需要在数据库中保存有限的时间。

在createIndex中指定expireAfterSeconds选项就可以创建一个TTL索引：

```
// 超时时间为24小时,默认是前台运行，可以通过background:true设置为后台模式
db.user_session.createIndex({"updated":1},{expireAfterSeconds:60*60*24});

```
<!--more-->

这样在updated字段上创建了一个TTL索引。如果一个文档的updated字段存在并且它的值是日期类型，当服务器时间比文档的updated字段的时间晚expireAfterSeconds秒时，文档就会被删除。

```
db.getCollection('user_session').insert(
  {
    _id: NumberInt(1),
    "updated":new Date(),
     username:'lisi'
  }
);

```
>mongodb保存时间使用的UTC时间，在查询出来的结果的时候会转换为GMT时间，所以你看到保存的时间和电脑时间相差8个小时(GMT+8)
>db.getCollection('user_session').find({updated:{$gt: new Date("2019-07-12 14:00:00")}})  在查询的时候可以使用new Date()直接进行时间的比较，new Date传入的参数是GMT时间

为了防止活跃的会话被删除，可以在会话上有活动发生时将updated字段的值更新为当前时间。只要updated的时间距离当前时间达到24小时。相应的文档就会被删除。

MongoDB的TTL功能依赖于mongodb中的后台线程，该线程读取索引中的日期类型值并从集合中删除过期的文档。
MongoDB每分钟对TTL索引进行一次清理，所以不应该依赖以秒为单位的时间保证索引的存活状态。而且TTL索引不保证在到期时立即删除过期数据。文档到期的时间与MongoDB从数据库中删除文档的时间之间可能存在延迟。由于删除过期文档的后台任务每60秒运行一次。所以，文档可能在文档到期和后台任务运行之间的期间保留在集合中。

>源码在 https://github.com/mongodb/mongo/blob/master/src/mongo/db/ttl.cpp

mongodb不支持使用createIndex来重新设置过期时间，只可以使用collMod命令修改expireAfterSeconds的值：

```
db.runCommand({collMod:"user_session",index: {name:"updated_1",expireAfterSeconds: 120}});
```

修改成功后，你会收到这样的消息(之前的过期时间是一分钟,现在修改为2分钟)

```
{
    "expireAfterSeconds_old" : 60.0,
    "expireAfterSeconds_new" : 120.0,
    "ok" : 1.0
}

```

在一个给定的集合上可以有多个TTL索引，你可以在created和updated字段分别建立ttl索引,但是不能同时使用两个字段建立复合ttl索引,也不能在同一个字段上又是创建TTL索引，又是创建普通索引，但是可以像“普通索引”一样用来优化排序和查询。