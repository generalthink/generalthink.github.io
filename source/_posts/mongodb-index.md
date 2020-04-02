---
title: 你知道MongoDB的10种索引吗?
date: 2020-04-01 14:09:17
tags: [MongoDB,索引]
---


### 为什么要有索引

查询快! 查询快！ 查询快！

### MongoDB的10种索引?

创建索引语法:
```
db.<collection_name>.createIndex( <key and index type specification>, <options> )
```

<!--more-->

我们的record Collection存在如下document

```
{
  "_id": ObjectId("570c04a4ad233577f97dc459"),
  "score": 1034,
  "userId": 123,
  "location": { state: "ZH", city: "ChengDu" },
  "addr": [
    {zip: "10036", detail: "高家村五组"},
    {zip: "94231", detail: "王家镇三组701"}
  ]
}

```

#### _id索引

mongodb会自动为document中的_id字段加上索引，所以能用_id查询就用_id查询

#### 单键索引
```
db.records.createIndex( { score: 1 })
```

#### 复合索引
```
db.records.createIndex( { userId: 1, score: 1})
```

#### 多值索引

```
db.records.createIndex( { "addr.zip": 1 })
```

#### 地理空间索引

MongoDB为坐标平面查询提供了专门的索引，称为地理空间索引。这种查询需要两个维度，所以参数是2d。
```
db.map.ensureIndex({"gps" : "2d"});
```
gps键的值必须是某种形式的一对值：一个包含2个元素的数组或者是包含2个键的内嵌文档
```
{"gps" : [0,100]}
{"gps" : {"x" : -30 , "y" : 30}}
{"gps" : {"latitude" : -180, "longitude" : 180 }}
```
至于键名可以随意。默认情况下，地理空间索引假设值范围是-180~180(对经纬度来说很方便),我们同样可以使用参数来对索引进行定制,比如下面的星图
```
db.star.trek.ensureIndex({"light-years" : "2d"} , {"min" : -1000, "max" : 1000, bits；10}， {collation: {locale: "simple"}});
```
上面的bits指定的是索引精度，默认情况下2d index使用的是26位精度，在默认范围-180~180中,大约等于60cm误差，最大可以设置32位精度。
索引精度不影响查询精度，降低精度的优点是插入操作的处理开销较低，并且占用的空间更少。较高精度的优点是查询扫描索引的较小部分以返回结果。

collation(排序规则)允许用户为字符串比较指定特定于语言的规则，例如字母和重音符号的规则。


地理空间的查询需要$near，它需要两个目标值的数组作为参数
```
db.map.find({"gps" : {"$near" : [40,-73]}}).limit(10);
```
默认查100个文档，如果不需要这么多，就应该设置一个少点的值以节约资源。

还可以使用
```
db.runCommand({geoNear : "map", near: [40,-73],num : 10})
```
geoNear的方式会返回每个文档到查询点的距离。

还可以查询矩形和圆形内所有的点,这个时候就要将原来的$near换成$geoWithin .
对于矩形要使用$box选项，它的参数是2个元素的数组，第一个元素指定了左下角坐标，第二个指定了右上角坐标

```
db.map.find({"gps" : {"$geoWithin " : {"$box" : [[10,20],[15,30]]}}});
```
如果要查询圆形，则要使用$center，参数变成了圆心和半径

```
db.map.find({"gps" : {"$geoWithin " : {"$center" : [[12,25],5]}}});
```


地理空间查询既可以使用平面几何，也可以使用球面几何，根据使用的查询和索引类型来决定。 2dsphere 索引只能支持球面几何，而 2d索引同时支持平面和球面几何。
然而，在 2dsphere索引上使用球面几何的查询将会更高效和准确。
> 2dsphere : https://docs.mongodb.com/manual/core/2dsphere/


它的应用场景可以是 查找附近美食，查找附近停车场等数据。

#### 全文索引

创建索引：

```
db.<collection_name>.createIndex({<key>: "text"});
```

查询数据：
```
db.<collection_name>.find( { $text: { $search: "green" } } );
```

查询以及排序:
```
db.<collection_name>.find(
{
 "$text": {
    "$search": "green"
  }
},
{
  "textScore": {
    "$meta": "textScore"
  }
}
).sort({
  "textScore": {
    "$meta": "textScore"
  }
})
```

注意这里的textScore并不是集合中的某个字段，而是mongodb根据搜索结果计算该条数据的分数(匹配度预告,值越大)


#### TTL索引

我在之前的文章讲到过这个[索引](https://juejin.im/post/5d4241d0e51d4561f95ee9c4)，它实际上是一个具有生命周期的索引,这种索引允许为每一个文档设置一个超时时间。一个文档达到预设置的老化程度后就会被删除。 

```
db.user_session.createIndex({"updated":1},{expireAfterSeconds:60*60*24});
```
如果一个文档的updated字段存在并且它的值是日期类型，当服务器时间比文档的updated字段的时间晚expireAfterSeconds秒时，文档就会被删除

```
db.getCollection('user_session').insert(
  {
    _id: NumberInt(1),
    "updated":new Date(),
     username:'lisi'
  }
);
```

#### 部分索引

```
db.<collection_name>.createIndex(
{'wechat': 1 },
{
  "partialFilterExpression": {
    "wechat": {
      "$exists": true
    }
  }
}
)

```

上面的索引表示对存在wechat字段的文档进行索引。部分索引仅索引集合中符合指定过滤器表达式的文档，降低了索引创建和维护的性能成本。

部分索引提供了稀疏索引功能的超集，应优先于稀疏索引使用


#### 稀疏索引

```
db.addresses.createIndex( { "xmpp_id": 1 }, { sparse: true } )
```
索引不索引不包含xmpp_id字段的文档。

#### 哈希索引

```
db.collection.createIndex( { _id: "hashed" })
```
哈希索引指按照某个字段的hash值来建立索引，目前主要用于MongoDB Sharded Cluster的Hash分片，hash索引只能满足字段完全匹配的查询，不能满足范围查询


### 后台方式创建索引

```
db.<collection_name>.createIndex( <key and index type specification>, {background: true})
```

建立索引即耗时也费力，还需要消耗很多资源。使用{background: true}选项可以使这个过程在后台完成，同时正常处理请求。要是不包括这个选项，数据库会阻塞建立索引期间的所有请求。
阻塞的做法会让索引建立得更快，同时也意味着应用在此期间不能应答。即便后台进行也会对正常操作有些影响，所以最好选在无关紧要的时刻。后台创建索引也会增加负载，好在不会让服务器宕机。

不过从4.2版本开始,所有索引构建都使用优化的构建过程，该过程仅在构建过程的开始和结束时才持有排他锁。其余的构建过程将产生交错的读写操作。如果指定该属性，MongoDB将忽略这个选项。