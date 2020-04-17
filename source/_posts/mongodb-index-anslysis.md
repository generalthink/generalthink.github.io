---
title: 索引优化-世人皆知Mysql,谁人懂我MongoDB
date: 2020-04-16 17:20:58
tags: [MongoDB]
---


### 查看执行计划

索引优化是一个永远都绕不过的话题,作为NoSQL的MongoDB也不例外。Mysql中通过explain命令来查看对应的索引信息,MongoDB亦如此。

```
1. db.collection.explain().<method(...)>
    db.products.explain().remove( { category: "apparel" }, { justOne: true })

2. db.collection.<method(...)>.explain({})
    db.products.remove( { category: "apparel" }, { justOne: true }).explain()

```

<!--more-->

如果你是在mongoshell 中第一种和第二种没什么区别，如果你是在robot 3T这样的客户端工具中使用你必须在后面显示调用finish()或者next()

```
db.collection.explain().find({}).finish()
```

explain有三种模式,分别是:

1. queryPlanner(默认) ：queryPlanner模式下并不会去真正进行query语句查询，而是针对query语句进行执行计划分析并选出winning plan
2. executionStats ：MongoDB运行查询优化器以选择获胜计划(winning plan)，执行获胜计划直至完成，并返回描述获胜计划执行情况的统计信息。
3. allPlansExecution: queryPlanner和executionStats都返回。相当于 `explain("allPlansExecution") = explain({})`


### queryPlanner(查询计划)

日志表中存储了用户的操作日志,我们经常查询某一篇文章的操作日志,数据如下:

```
{
  "_id" : NumberLong(7277744),
  "operatorName" : "autotest_cp",
  "operateTimeUnix" : NumberLong(1586511800890),
  "module" : "ARTICLE",
  "opType" : "CREATE",
  "level" : "GENERAL",
  "recordData" : {
      "articleId" : "6153324",
      "categories" : "100006",
      "title" : "testCase-2 this article is created for cp edior to search",
      "status" : "DRAFT"
  },
  "responseCode" : 10002
}

```

集合中大概有700万数据,对于这样的查询语句
```
db.getCollection('operateLog').find({"module": "ARTICLE", "recordData.articleId": "6153324"}).sort({_id:-1})
```

首先看下queryPlanner返回的内容：

```json
"queryPlanner" : {
  "plannerVersion" : 1,
  "namespace" : "smcp.operateLog",
  "indexFilterSet" : false,
  "parsedQuery" : {
    "$and" : [ 
      {
        "module" : {
            "$eq" : "ARTICLE"
        }
      }, 
      {
        "recordData.articleId" : {
            "$eq" : "6153324"
        }
      }
    ]
  },
  "winningPlan" : {
    "stage" : "FETCH",
    "filter" : {
      "$and" : [ 
        {
          "module" : {
              "$eq" : "ARTICLE"
          }
        }, 
        {
          "recordData.articleId" : {
              "$eq" : "6153324"
          }
        }
      ]
    },
    "inputStage" : {
      "stage" : "IXSCAN",
      "keyPattern" : {
          "_id" : 1
      },
      "indexName" : "_id_",
      "isMultiKey" : false,
      "multiKeyPaths" : {
          "_id" : []
      },
      "isUnique" : true,
      "isSparse" : false,
      "isPartial" : false,
      "indexVersion" : 2,
      "direction" : "backward",
      "indexBounds" : {
          "_id" : [ 
              "[MaxKey, MinKey]"
          ]
      }
    }
  },
  "rejectedPlans" : [ 
    {
      "stage" : "SORT",
      "sortPattern" : {
          "_id" : -1.0
      },
      "inputStage" : {
        "stage" : "SORT_KEY_GENERATOR",
        "inputStage" : {
          "stage" : "FETCH",
          "filter" : {
              "recordData.articleId" : {
                  "$eq" : "6153324"
              }
          },
          "inputStage" : {
            "stage" : "IXSCAN",
            "keyPattern" : {
                "module" : 1.0,
                "opType" : 1.0
            },
            "indexName" : "module_1_opType_1",
            "isMultiKey" : false,
            "multiKeyPaths" : {
                "module" : [],
                "opType" : []
            },
            "isUnique" : false,
            "isSparse" : false,
            "isPartial" : false,
            "indexVersion" : 2,
            "direction" : "forward",
            "indexBounds" : {
                "module" : [ 
                    "[\"ARTICLE\", \"ARTICLE\"]"
                ],
                "opType" : [ 
                    "[MinKey, MaxKey]"
                ]
            }
          }
        }
      }
    }
  ]
}
```

#### 字段含义

一些重要字段的含义

+ queryPlanner.namespace
查询的哪个表

+ queryPlanner.winningPlan
查询优化器针对该query所返回的最优执行计划的详细内容。

+ queryPlanner.winningPlan.stage
最优计划执行的阶段,每个阶段都包含特定于该阶段的信息。例如，IXSCAN阶段将包括索引范围以及特定于索​​引扫描的其他数据。如果一个阶段具有一个子阶段或多个子阶段，那么该阶段将具有inputStage或inputStages。

+ queryPlanner.winningPlan.inputStage
描述子阶段的文档，该子阶段向其父级提供文档或索引键。如果父阶段只有一个孩子，则该字段存在。

+ queryPlanner.winningPlan.inputStage.indexName
winning plan所选用的index,这里是根据_id来进行排序的，所以使用了_id的索引

+ queryPlanner.winningPlan.inputStage.isMultiKey
是否是Multikey，此处返回是false，如果索引建立在array上，此处将是true

+ queryPlanner.winningPlan.inputStage.isUnique
使用的索引是否是唯一索引，这里的_id是唯一索引

+ queryPlanner.winningPlan.inputStage.isSparse
是否是稀疏索引

+ queryPlanner.winningPlan.inputStage.isPartial
是否是部分索引

+ queryPlanner.winningPlan.inputStage.direction 
此query的查询顺序，默认是forward，由于使用了sort({_id:-1})显示backward

+ queryPlanner.winningPlan.inputStage.indexBounds
winningplan所扫描的索引范围,由于这里使用的是sort({_id:-1}),对_id倒序排序,所以范围是[MaxKey,MinKey]。如果是正序,则是[MinKey,MaxKey]

+ queryPlanner.rejectedPlans
拒绝的计划详细内容,各字段含义同winningPlan

### executionStats(执行结果)

再来看下executionStats的返回结果

```
"executionStats" : {
  "executionSuccess" : true,
  "nReturned" : 1,
  "executionTimeMillis" : 24387,
  "totalKeysExamined" : 6998084,
  "totalDocsExamined" : 6998084,
  "executionStages" : {
    "stage" : "FETCH",
    "filter" : {
      "$and" : [ 
        {
          "module" : {
              "$eq" : "ARTICLE"
          }
        }, 
        {
          "recordData.articleId" : {
              "$eq" : "6153324"
          }
        }
      ]
    },
    "nReturned" : 1,
    "executionTimeMillisEstimate" : 1684,
    "works" : 6998085,
    "advanced" : 1,
    "needTime" : 6998083,
    "needYield" : 0,
    "saveState" : 71074,
    "restoreState" : 71074,
    "isEOF" : 1,
    "invalidates" : 0,
    "docsExamined" : 6998084,
    "alreadyHasObj" : 0,
    "inputStage" : {
      "stage" : "IXSCAN",
      "nReturned" : 6998084,
      "executionTimeMillisEstimate" : 290,
      "works" : 6998085,
      "advanced" : 6998084,
      "needTime" : 0,
      "needYield" : 0,
      "saveState" : 71074,
      "restoreState" : 71074,
      "isEOF" : 1,
      "invalidates" : 0,
      "keyPattern" : {
          "_id" : 1
      },
      "indexName" : "_id_",
      "isMultiKey" : false,
      "multiKeyPaths" : {
          "_id" : []
      },
      "isUnique" : true,
      "isSparse" : false,
      "isPartial" : false,
      "indexVersion" : 2,
      "direction" : "backward",
      "indexBounds" : {
          "_id" : [ 
              "[MaxKey, MinKey]"
          ]
      },
      "keysExamined" : 6998084,
      "seeks" : 1,
      "dupsTested" : 0,
      "dupsDropped" : 0,
      "seenInvalidated" : 0
    }
  },

  "allPlansExecution" : [
    {...},
    {...}
  ]
}
```

#### 字段解析

+ executionStats.executionSuccess
是否执行成功

+ executionStats.nReturned
查询的返回条数

+ executionStats.executionTimeMillis
选择查询计划和执行查询所需的总时间（以毫秒为单位）

+ executionStats.totalKeysExamined
索引扫描次数

+ executionStats.totalDocsExamined
document扫描次数

+ executionStats.executionStages
以阶段树的形式详细说明获胜计划的完成执行情况；即一个阶段可以具有一个inputStage或多个inputStages。如上说明。

+ executionStats.executionStages.inputStage.keysExamined
扫描了多少次索引

+ executionStats.executionStages.inputStage.docsExamined
扫描了多少次文档，一般当stage是 COLLSCAN的时候会有这个值。

+ exlexecutionStats.allPlansExecution
这里展示了所有查询计划的详细。(winningPlan + rejectPlans),字段含义和winningPlan中一致，不做赘述


### 15种stage

从上面可以看出stage是很重要的，一个查询到底走的是索引还是全表扫描主要看的就是stage的值, 而stage有如下值

1. COLLSCAN : 扫描整个集合
2. IXSCAN : 索引扫描(index scan)
3. FETCH : 根据索引返回的结果去检索文档(如上我们的例子)
4. SHARD_MERGE : 将各个分片返回数据进行merge
5. SORT : 调用了sort方法,当出现这个阶段的时候你可以看到memUsage以及memLimit这两个字段
6. SORT_KEY_GENERATOR : 在内存中进行了排序
6. LIMIT ： 使用limit限制返回数
7. SKIP ： 使用skip进行跳过
8. IDHACK ： 针对_id进行查询
9. SHARDING_FILTER ：通过mongos对分片数据进行查询
10. COUNT: 利用db.coll.explain().count()之类进行count运算, 只要调用了count方法，那么 executionStats.executionStages.stage = COUNT
11. COUNT_SCAN : count使用Index进行count时的stage返回

```
{
  country: "ID",
  name: "jjj",
  status: 0
},
{
  country: "ZH",
  name: "lisi",
  status: 1
}
```

我们对country字段建立了索引，同时执行下面的语句
```
db.getCollection('testData').explain(true).count({country: "ID"})
```
那么查看执行结果可以看到 executionStats.executionStages.inputStage.stage = COUNT_SCAN, COUNT_SCAN是COUNT的一个子阶段。


12. COUNTSCAN : count不使用Index进行count时的stage返回。

```
db.getCollection('testData').explain(true).count({status: 0})
```

此时 executionStats.executionStages.inputStage.stage = COUNTSCAN , COUNTSCAN是COUNT的一个子阶段


13. SUBPLAN : 未使用到索引的$or查询的stage返回
```
db.getCollection('testData').find({$or : [{name : "lisi"}, {status: 0}]}).explain(true);
```
此时 executionStats.executionStages.stage = SUBPLAN

14. TEXT : 使用全文索引进行查询时候的stage返回
15. PROJECTION : 限定返回字段时候stage的返回


查看executionStats.executionStages.stage以及其下各个inputStage(子阶段)的值是什么,可以判定存在哪些优化点。

**一个查询它扫描的文档数要尽可能的少,才能更快，明显我们我们不希望看到COLLSCAN, SORT_KEY_GENERATOR, COUNTSCAN, SUBPLAN 以及不合理的 SKIP 这些stage,当你看到这些stage的时候就要注意了。**



### 查询优化

当你看winningPlan或者rejectPlan的时候，你就可以知道执行顺序是怎样的，比如我们rejectPlan中，先是通过 "module_1_opType_1"检索 "module = ARTICLE"的数据，然后FETCH阶段再通过 "recordData.articleId=6153324"进行过滤，最后在内存中排序后返回数据。 明显这样的计划被拒绝了，至少它没有winningPlan执行快。

再来看看executionStats返回的数据
**
nReturned 为 1，即符合条件的只有1条
executionTimeMillis 值为24387,执行时间为24秒
totalKeysExamined 值为 6998084,虽然用到了索引，但是几乎是扫描了所有的key
totalDocsExamined的值为6998084,也是扫描了所有文档
**

从上面的输出结果可以看出来，虽然我们使用了索引，但是速度依然很慢。很明显现在的索引，并不适合我们，为了排除干扰，我们先将module_1_opType_1这个索引删除。由于我们这里使用了两个字段进行查询，而 recordData.articleId这个字段并不是每个document(集合中还存储了其他类型的数据)都存在，所以建立索引的时候recordData.articleId需要建立部分索引

```
db.getCollection('operateLog').createIndex(
{'module': 1, 'recordData.articleId': 1 },
{
  "partialFilterExpression": {
    "recordData.articleId": {
      "$exists": true
    }
  },

  "background": true
}
)

```

我先吃个苹果，等它把索引建立好，大家有啥吃啥。在索引建立完成之后，我们来看看 executionStats 的结果

```
"executionStats" : {
  "executionSuccess" : true,
  "nReturned" : 1,
  "executionTimeMillis" : 3,
  "totalKeysExamined" : 1,
  "totalDocsExamined" : 1,
  "executionStages" : {
    "stage" : "SORT",
    "sortPattern" : {
        "_id" : -1.0
    },
    "memUsage" : 491,
    "memLimit" : 33554432,
    "inputStage" : {
      "stage" : "SORT_KEY_GENERATOR",
      "inputStage" : {
          "stage" : "FETCH",
          "nReturned" : 1,
          "inputStage" : {
            "stage" : "IXSCAN",
            "keyPattern" : {
                "module" : 1.0,
                "recordData.articleId" : 1.0
            },
            "indexName" : "module_1_recordData.articleId_1",
            "isMultiKey" : false,
            "multiKeyPaths" : {
              "module" : [],
              "recordData.articleId" : []
            },
            "isPartial" : true,
            "indexVersion" : 2,
            "direction" : "forward",
            "indexBounds" : {
              "module" : [ 
                "[\"ARTICLE\", \"ARTICLE\"]"
              ],
              "recordData.articleId" : [ 
                "[\"6153324\", \"6153324\"]"
              ]
          }
        }
      }
    }
  }
}
```

我忽略了一些不重要的字段,可以看到，**现在执行时间是3毫秒(executionTimeMillis=3),扫描了1个index(totalKeysExamined=1),扫描了1个document(totalDocsExamined=1)。相比于之前的24387毫秒，我可以说我的执行速度提升了8000倍，我就问还有谁。**如果此事让UC 震惊部小编知道了，肯定又可以起一个震惊的标题了


**但是这个执行计划仍然有问题，有问题，有问题，重要的事情说三遍。 executionStages.stage = sort,证明它在内存中排序了,在数据量大的时候，是很消耗性能的，所以千万不能忽视它，我们要改进这个点。**

我们要在 nReturned = totalDocsExamined的基础上，让排序也走索引。所以我们先将之前的索引删除，然后重新创建索引，这里我们将_id字段也加入到索引中，三个字段形成组合索引

```
db.getCollection('operateLog').createIndex(
{'module': 1, 'recordData.articleId': 1, '_id': -1},
{
  "partialFilterExpression": {
    "recordData.articleId": {
      "$exists": true
    }
  },

  "background": true
}
)

```

同样的再来看看我们的执行结果:

```
"executionStats" : {
  "executionSuccess" : true,
  "nReturned" : 1,
  "executionTimeMillis" : 0,
  "totalKeysExamined" : 1,
  "totalDocsExamined" : 1,
  "executionStages" : {
    "stage" : "FETCH",
    "nReturned" : 1,
    "executionTimeMillisEstimate" : 0,
    "docsExamined" : 1,
    "inputStage" : {
      "stage" : "IXSCAN",
      "nReturned" : 1,
      "keyPattern" : {
          "module" : 1.0,
          "recordData.articleId" : 1.0,
          "_id" : -1.0
      },
      "indexName" : "module_1_recordData.articleId_1__id_-1",
      "multiKeyPaths" : {
          "module" : [],
          "recordData.articleId" : [],
          "_id" : []
      },
      "isPartial" : true,
      "direction" : "forward",
      "indexBounds" : {
          "module" : [ 
              "[\"ARTICLE\", \"ARTICLE\"]"
          ],
          "recordData.articleId" : [ 
              "[\"6153324\", \"6153324\"]"
          ],
          "_id" : [ 
              "[MaxKey, MinKey]"
          ]
      }
    }
  }
}
```

可以看到我们这次的stage是FETCH+IXSCAN,同时 nReturned = totalKeysExamined = totalDocsExamined = 1，并且利用了index排序，而非在内存中排序。从executionTimeMillis=0也可以看出来，性能相比于之前的3毫秒也有所提升，至此这个索引就是我们需要的了。


最开头的结果和优化的过程告诉我们,使用了索引你的查询仍然可能很慢,我们要将更多的目光集中到扫描的文档或者行数中。

### 索引优化准则

1. 根据ESR原则创建索引
精确(Equal)匹配的字段放最前面,排序(Sort)条件放中间,范围(Range)匹配的字段放最后面,同样适用于ES,ER。

2. 每一个查询都必须要有对应的索引

3. 尽量使用覆盖索引 Covered Indexes(可以避免读数据文件)
  需要查询的条件以及返回值均在索引中

4. 使用 projection 来减少返回到客户端的的文档的内容

5. 尽可能不要计算总数,特别是数据量大和查询不能命中索引的时候

6. 避免使用skip/limit形式的分页，特别是数据量大的时候

  替代方案:使用查询条件+唯一排序条件
  第一页：`db.posts.find({}).sort({_id: 1}).limit(20);`
  第二页：`db.posts.find({_id: {$gt: <第一页最后一个_id>}}).sort({_id: 1}).limit(20);`
  第三页：`db.posts.find({_id: {$gt: <第二页最后一个_id>}}).sort({_id: 1}).limit(20);`