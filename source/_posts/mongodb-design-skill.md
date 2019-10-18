---
title: MongoDB设计技巧
date: 2019-10-18 15:01:56
tags: MongoDB
---

### 范式化设计还是反范式

考虑下这样的场景，我们的订单数据是这样的

```
商品：
{
  "_id": productId,
  "name": name,
  "price": price,
}

订单:
{
  "_id": orderId,
  "user": userId,
  "items": [
    productId1,
    productId2,
    productId3
  ]
}
```
<!--more-->

当我们查询订单内容的时候，先通过orderId查询订单，然后在通过订单信息中的productId查询到对应的商品信息。这种设计下一次查询无法获取完整的订单。

范式化结果就是读取速度比较忙，当所有订单的一致性会有保证。

在来看看反范式化设计

```
订单:
{
  "_id": orderId,
  "user": userId,
  "items": [
   {
    "_id": productId1,
    "name": name,
    "price": price,
   },
   {
    "_id": productId2,
    "name": name,
    "price": price,
   },
  ]
}

```

这里将商品信息作为内嵌文档存在订单数据中，这样当显示的时候就只需要一次查询就可以了。

反范式读取速度快，一致性稍弱，商品信息的变更不能原子性地更新到多个文档。

那么我们一般使用哪一个呢？我们在设计的时候要考虑以下问题


1. 读写比是怎样的？

可能读取了商品信息一万次才修改一次它的详细信息，为了那一次写入快一点或者保证一致性，搭上一万次的读取消耗值得吗？还有你认为引用的数据多久会更新一次？更新越少，越适合反范式化。有些极少变化的数据几乎根本不值得引用。比如名字，性别，地址等。

2. 一致性重要吗？

如果是肯定的，则应该范式化。

3. 要不要快速的读取？
如果想要读取尽可能快，则要反范式化。在这个引用中就无所谓了，所以不能算考量因素，实时的应用要尽可能地反范式化。

订单文档非常适合反范式化，因为其中的商品信息不经常变化。就算变了也不必更新到所有订单。范式化再次就没有什么优势可言了。

所以本例中就是将订单反范式化。



### 嵌入时间点数据

当一个商品打折或者换了图片，并不需要更改原来的订单中的信息。类似这种特定于某一时刻的时间点数据，都应该做嵌入处理。

在我们上面提到的订单文档中有一处也是这样，地址就属于时间点数据。若某人更新了个人信息，那么并不需要改变其以往的订单内容。

### 千万不要嵌入不断增加的数据

MongoDB存储数据的机制决定了对数组不断追加数据是很低效的。在正常使用中数组和对象大小应该相对固定。

嵌入20，100，或者100000个子文档都不是问题，关键是提前这么做，之后基本保持不变。否则放任文档增长会使得系统慢的你受不了。

对于那些不断增加的内容，必须评论这个时候应该将其作为单独的文档处理比较合适。

### 尽可能预先分配空间

只要知道文档开始比较小，后来会变为确定的大小就可以使用这种优化方法，一开始插入文档的时候，就用和最终数据大小一样的垃圾数据填充，比如添加一个garbage字段(其中包含一个字符串，串大小与文档最终大小相同)，然后马上重置字段
```
db.collection.insert({"_id" : 1,/* other fields */, "garbase": longString});
db.collection.update({"_id" : 1, });

```
这样，MongDB就会为文档今后的增长分配足够的空间

>mongodb中存储文档是预留了空间的，允许文档扩容，但是当文档增大到一定地步的时候，就会超过原本分配的空间，此时文档就会进行移动

### 用数组存放要匿名访问的内嵌数据

一个常见的问题就是内嵌的信息到底是用数组还是用子文档存。如果确切知道要查询的内容，就要用子文档。如果有时候不太清楚查询的具体内容，就要用数组。当知道一些条目的查询条件时，通常该使用数组。

假设我想记录下游戏中某些物品的属性。我们可以这样建模

```
{
  "_id": 1,
  "items" : {

    "slingshot": {
      "type" : "weapon",
      "damage" : 30,
      "ranged" : true
    },

    "jar" : {
      "type": "container",
      "contains": "fairy"
    }

  }
}

```
假设要找出所有damage大于20的武器，子文档不支持这种查找方式，你只能知晓具体某种物品的信息才能查找，比如{"items.jar.damage": {"$gt":20}}.
如果无需标识符，就要用数组
```
{
  "_id": 1,
  "items" : [

    {
      "id" : "slingshot"
      "type" : "weapon",
      "damage" : 30,
      "ranged" : true
    },

    {
      "id" : "jar",
      "type": "container",
      "contains": "fairy"
    }

  ]
}

```

比如{”items.damage":{"$gt":20}}就行了。如果还需要多条件查询，可以使用$elemMatch.

### 如何使用自增id代替ObjectId

有时候在使用过程中受限于业务或者其他情况，并不想使用ObjectId,而是想要使用自动Id来代替。但是MongoDB本身并没有提供这个功能，那么如何实现呢？

可以新建一个collection来保存自增id

```
{
    "_id" : ObjectId("59ed8d3df772d09a67eb25f6"),
    "fieldName" : "user",
    "seq" : NumberLong(100064)
}

```
fieldName表示哪个集合,那么下次要使用的时候只用取出这个值加1就可以了。代码如下

```java
 public Long getNextSequence(String fieldName, long gap) {
    try {
        Query query = new Query();
        query.addCriteria(Criteria.where("fieldName").is(fieldName));

        Update update = new Update();
        update.inc("seq", gap);

        FindAndModifyOptions options = FindAndModifyOptions.options();
        options.upsert(true);
        options.returnNew(true);

        Counter counter = mongoTemplate.findAndModify(query, update, options, Counter.class);

        if (counter != null) {
            return counter.getSeq();
        }
    } catch (Throwable t) {
        log.error("Exception when getNextSequence from mongodb", t);
    }
    return gap;
}

```



### 不要到处使用索引

索引是很强大，但是要提醒你的是，不是所有查询都可以用索引的。比如你要返回集合中90%的文档而非获取一些记录，就不应该使用索引。

如果对这种查询用了索引，结果就是几乎遍历整个索引树，把其中一部分，比方说40GB的索引都加载到内存。然后按照索引中的指针加载集合中200GB的文档数据，最终将加载
200GB + 40GB = 240GB的数据，比不用索引还多。

所以索引一般用在返回结果只是总体数据的小部分的时候。根据经验，一旦要大约返回集合一般的数据就不要使用索引了。

若是已经对某个字段建立了索引，又想在大规模查询时不使用它(因为使用索引可能会较低效),可以使用自然排序来强制MongoDB禁用索引。自然排序就是“按照磁盘上的存储顺序返回数据",这样MongDB就不会使用索引了。

```
db.students.find().sort({"$natural" : 1});
```

如果某个查询不用索引，MongoDB就会做全表扫描。

### 索引覆盖查询

如果你想要返回某些字段且这些字段都可以放到索引中，那么MongoDB可以做索引覆盖查询，这种查询不会访问指针指向的文档，而是直接用索引的数据返回结果，比如有如下索引

```
db.students.ensureIndex(x : 1, y : 1, z : 1);
```

现在查询被索引的字段，并要求只返回这些字段，MongoDB就没有必要加载整个文档

```
db.students.find({"x" : "xxx", "y" : "xxx"},{x : 1, y : 1, z : 1, "_id" : 0});
```

注意由于_id是默认返回的，而它又不是索引的一部分，所以MongoDB就需要到文档中获取_id,去掉它，就可以仅根据索引返回结果了。

若是查询值返回几个字段，则考虑将其放到索引中，即使不对他们执行查询，也能做索引覆盖查询。比如上面的字段z。

### AND查询要点

假设要查询满足条件A、B、C的文档。满足A的文档有40000，满足B的有9000，满足C的有200，要是让MongoDB按照这个顺序查询，效率可不高。

如果把C放到最前，然后是B，然后是A，则针对B,C只需要查询最多200个文档。

这样工作量显著减少了。要是已知某个查询条件更加苛刻，则要将其放置到最前面。

### OR型查询要点

OR与AND查询相反，匹配最多的查询语句应该放到最前面，因为MongDB每次都要匹配不在结果集中的文档。

### 单表查询尽量使用Respostory

开发中，对于简单的查询我一般使用MongoRepository来实现功能，如果有复杂的结合MongoTemplate，注意这两者是可以混合使用的。

### converter的建议

开发中我们要写对于一个collection,其中有些特殊的类型(比如枚举)需要我们写converter,大多时候是双向的，比如db-->collection和collection-->db
如果只有一个类型需要转换，我们可以针对这一个属性进行转换，比如下面的例子

```java

@WritingConverter
@Component
public class UserStatusToIntConverter implements Converter<UserStatus, Integer> {

    @Override
    public Integer convert(UserStatus userStatus) {
        return userStatus.getStatus();
    }
}


@ReadingConverter
@Component
public class UserStatusFromIntConverter implements Converter<Integer, UserStatus> {

    @Override
    public UserStatus convert(Integer source) {
        return UserStatus.findStatus(source);
    }
}

```

一个字段还好，如果一个类中有很多个字段都需要做转换的话，就会产生很多个converter,这个时候我们可以写一个类级别的转换器

```java
@ReadingConverter
@Component
public class OperateLogFromDbConverter extends AbstractReadingConverter<Document, OperateLog> {
  @Override
  public OperateLog convert(Document source) {

      OperateLog opLog = convertBasicField(source);

      if (source.containsKey("_id")) {
          opLog.setId(source.getLong("_id"));
      }

      if (source.containsKey("module")) {

          opLog.setModule(ModuleEnum.findModule(source.getInteger("module")));
      }

      if (source.containsKey("opType")) {
          opLog.setOpType(OpTypeEnum.findOpType(source.getInteger("opType")));
      }

      if (source.containsKey("level")) {
          opLog.setLevel(OpLevelEnum.findOpLevel(source.getInteger("level")));
      }

      return opLog;
  }

  private OperateLog convertBasicField(Document source) {
      Gson gson = new Gson();
      return gson.fromJson(source.toJson(), OperateLog.class);
  }
}

```

上面代码我用了GSON做common field的转换，如果你不这样写，就需要判断每个字段，然后进行填充。
