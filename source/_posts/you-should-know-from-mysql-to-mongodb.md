title: 从mysql到mongodb你应该快速了解的内容
date: 2018-08-05 20:19:43
tags: mongodb
categories: mongodb
description: 从关系型数据库突然转战到Mongodb最开始很是不习惯，但是好在官网的文档写得也挺详细的,一个数据库最重要的什么？当然是增删改查了，所以只要掌握了增删改查就可以解决平时80%的问题了，本文就抛砖引玉快速的学习下mongodb的使用。
keywords: mongodb,mongoimport,mongoexport
---


从mysql转战mongodb，应该快速的了解上手，以下的内容就是个人对其基本使用的一个总结。

### 下载安装查看官方文档

当然第一步肯定是要安装上了，安装步骤按照官网的来就可以了  <https://docs.mongodb.com/manual/mongo/>,安装之后可以使用自带的mongoshell来测试也可以用Robo 3T这样的工具来进行测试

### 一些基本概念

MongoDB以BSON格式的文档（Documents）形式存储。Databases中包含集合（Collections），集合（Collections）中存储文档（Documents）。

[BSON](https://docs.mongodb.com/manual/reference/bson-types/)是一个二进制形式的JSON文档，它比JSON包含更多的数据类型。


| SQL术语/概念  | MongoDB术语/概念  | 解释/说明  |
|---|---|---|
| database  |  database |  数据库 |
|  table |  collection | 数据库表/集合  |
| row  | document  |  数据记录行/文档 |
|  column | field  | 数据字段/域  |
|  index | index  | 索引  |
|  table joins |   |  table joins |
|  primary key | primary key  | 主键,MongoDB自动将_id字段设置为主键  |

上面说了MongoDB中保存的数据是bson格式的数据,所以数据库中的数据看上去是这样的

```json
{
    "_id" : 1,
    "zipcode" : "63109",
    "students" : [
        {
            "name" : "john",
            "age" : 10
        },
        {
            "name" : "jess",
            "age" : 11
        },
        {
            "name" : "jeff",
            "age" : 15
        }
    ]
}
```

### 熟悉语法

#### 连接数据库

```
mongo host:port/databaseName -u user -p password

#连接本地的local库
mongo 127.0.0.1/local

```

#### 插入数据

```
use myNewDB

#也可以调用insertMany同时插入多条数据,插入的数据是一个/多个json对象
db.myNewCollection1.insert( {name:"Kit"} )

# 也以使用getCollection的方式
db.getCollection("myNewCollection1").insertOne({name:"Kit"});

```
insert()操作会创建名为myNewDB的database和名为myNewCollection1的collection（如果他们不存在的话）。


#### 查询数据

```
db.collectionName.find(query,projection);

query ：可选，使用查询操作符指定查询条件
projection ：可选，使用投影操作符指定返回的键。查询时返回文档中所有键值， 只需省略该参数即可（默认省略）。

```

##### query的使用

下面给出几个对比查询

```
# select * from user where name = "张三" and age = 20;
db.user.find({
    name:"张三",
    age:20
});


# select * from user where name = "张三" and age > 20;
db.user.find({
    name:"张三",
    age:{
        $gt:20
    }
});


# select * from user where name = "张三" or age = 20;
db.user.find({
    $or: [
        {name:"张三"},
        {age:20}
    ]
});

```

基本mysql支持的查询，mongodb都可以做到，它支持的操作符在这里(<https://docs.mongodb.com/manual/reference/operator/query/>)可以看到.

从上面的对比基本可以总结出来，mongodb的查询基本是按照JSON对象的语法来进行的,所以想要查询数据就按照JSON对象的方式来，所以你能怎么访问就能怎么查询，比如对嵌套对象或者数据对象的数据。

**对内嵌对象的查询**

如果存在这样的数据:
```

db.user.insert(
{
    name:"张三",
    age:20,
    sex:"男",
    sale_books:[
        {
            name:"钢铁怎么练成的",
            price:69
        },
        {
            name:"Spring揭秘",
            price:87
        },
        {
            name:"MongoDb实战",
            price:54
        }
    ],
     address: {
         zipCode: 632185,
         name: "腾讯大学287号"
     }
}

);


```
我现在想要查询MongoDb实战这本书的信息，怎么查？

```
db.user.find({
    "sale_books.name" : "MongoDb实战"
});
```
但是运行这条sql语句会返回sale_books数据中其他的数据，如何只返回MongoDb实战的数据呢？这个时候可以考虑使用projection的方式,那么下面这种就可以满足需求

```

db.user.find(
{
    "sale_books.name" : "MongoDb实战"
},
{
    # $用于遍历数组的下标
    "sale_books.$":1
});

或者这样

db.user.find(
sale_books: {
  $elemMatch: {
   name: "MongoDb实战"
  }
}
},
{
  "sale_books.$":1
}
)


```



看上去和访问json对象的方式是一样的，没错这就是mongodb的查询方式。

更多的关于数据对象的增删改查可以参考这篇文章(<https://blog.csdn.net/leshami/article/details/55192965>).

关于query的用法基本可以总结为以下三种:

```
1. equal的查询语法: { <field1>: <value1>, ... },比如db.inventory.find( { status: "D" } )

2. 使用查询操作符的查询语法: { <field1>: { <operator1>: <value1> }, ... },比如db.inventory.find( { status: { $in: [ "A", "D" ] } } )

3. {<operator1> : { <field1>: { <operator1>: <value1> }}},比如or的用法

```


##### projection的用法 #####

```

# inclusion模式 指定返回的键，不返回其他键
db.collection.find(query, {name: 1, age: 1})

# exclusion模式 指定不返回的键,返回其他键,默认会返回_id,如果不想返回需要手动指定
db.collection.find(query, {name: 0, age: 0})

# 两种模式不可混用（因为这样的话无法推断其他键是否应返回）
# 错误
db.collection.find(query, {title: 1, by: 0})


# 相当于select name from user where age = 20;
db.user.find({age:20},{name:1});

```

#### 修改数据

常用语法格式如下：

```
db.collection.update(<filter>, <update>, <options>)
db.collection.updateOne(<filter>, <update>, <options>)
db.collection.updateMany(<filter>, <update>, <options>)
db.collection.replaceOne(<filter>, <update>, <options>)

```

可以看到关于update的操作主要有三个参数，第一个是查询条件，第二个是需要更新的内容，第三个是一些特殊选项，update的语法也是json格式(记住所有的语法都是json格式)

```
{
  <update operator>: { <field1>: <value1>, ... },
  <update operator>: { <field2>: <value2>, ... },
  ...
}

```

演示一个例子，如果我想要实现这样的sql语句,

```
update user set age = age + 1 where name = '张三'

```

那么对应的mongo查询可以这样写：

```
db.getCollection('user').updateOne(
{
    name:"张三"
},
{
    $inc:
        {
            age:1
        }

}
);

```

第三个参数有时候也可能用到，他们的参数如下：
```
{
     upsert: <boolean>,
     multi: <boolean>,
     writeConcern: <document>
}

upsert : 可选，这个参数的意思是，如果不存在update的记录，是否插入update指定的对象,true为插入，默认是false，不插入。
multi : 可选，mongodb 默认是false,只更新找到的第一条记录，如果这个参数为true,就把按条件查出来多条记录全部更新。
writeConcern :可选，抛出异常的级别。

```

如果想要给一条记录新增一个sex的对象，那么可以这样写

```
db.user.update({name:"张三"},{$set:{sex:"male"}},{upsert:true})
```

update支持的操作符在这里可以看到(<https://docs.mongodb.com/manual/reference/operator/update-field/>)

#### 删除数据

删除数据的方法主要有2个,它们的语法格式也很简单，是按照query的语法来进行的，所以删除数据的语法可以参考查询数据那一节。

```
db.collection.deleteMany()
db.collection.deleteOne()

```


#### 备份数据

导出只能在本地命令行执行而不是mongo shell中执行，所以不用用mongo先进入到命令行界面，如果时导出本地数据则不用添加-h参数

```
mongoexport -h host:port -u username -p password -d databaseName -c collectionName -o exportFilePath --type json/csv -f field

参数说明:
    -h : 主机地址
    -u : 用户名
    -p ：密码
    -d ：数据库名
    -c ：collection名
    -o ：输出的文件名
    --type ： 输出的格式，默认为json，可选为CSV
    -f ：输出的字段，如果-type为csv，则需要加上-f "字段名"

mongoexport -h localhost:27017

```


#### 导入数据

```
mongoimport -h host:port -u username -p password  -d dbname -c collectionname --file filename --headerline --type json/csv -f field

参数说明：
    -h : 主机地址
    -d ：数据库名
    -c ：collection名
    --type ：导入的格式默认json
    -f ：导入的字段名
    --headerline ：如果导入的格式是csv，则可以使用第一行的标题作为导入的字段
    --file ：要导入的文件

mongoimport -h localhost:27017 -d local-test -c userSetting --file E:/userSetting.json --type json

```

#### 执行脚本

在mysql里面我可以用mysql命令直接运行脚本，在mongo里面我们同样可以，只是mongo里面的脚本就是js,这里面你可以用上面用到的任何一个方法，insert,find,update,delete之类的，这个脚本里面就是js语法，js的一些特性也是可以使用的。比如我想向不同国家的user中插入一条数据

user.js文件内容如下：

```js
function insertUser(country) {
    var collectionName = "user_info";
    var userData = db.getCollection(collectionName).find({"country":country});
    var desc = "Hello, my country is " + country;
    db.getCollection(collectionName).insert({"conuntry":country,"desc":desc});
}
["CN","UK"].forEach(function(c) {insertUser(c)});

```

在命令行执行

```
mong host:port/databaseName filePath.js

```

即可,或者在mongo shell中采用load的方式执行即可，需要注意的是如果语法有错误可能会说报file doesn't exist的问题。这个时候可以加上--quiet的方式会有一些错误提示。如果想查看更多的参数使用mongo --help的方式查看
