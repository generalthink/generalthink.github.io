---
title: deepseek真有这么强吗？
date: 2025-02-10 10:47:02
tags: [deepseek, AI]
---


DeepSeek喧嚣尘土，你常常可以看到DeepSeek多么牛逼，怎样怎样之类的文章。

但是DeepSeek真有那么强吗？在没有亲自体验过的时候，我不禁要打一个问号,所以我分别问了它(deepseek-r1)和Claude(Claude-V3.5-sonnet模型)4个问题，来看看到底哪个更强?


### 测试问题

我分别从生活和工作方面提问了4个问题

#### 问题1

> 我有一个朋友，和她相恋4年的女朋友因为彩礼问题闹崩了， 现在分手了，他很伤心，还是放不下， 我应该怎么安慰他？


![cluade的答案](/images/ai/cluade-answer-1.png)


![deepseek的答案](/images/ai/deepseek-answer-1.png)

看起来不相伯仲，deepseek的答案在这里并不突出。

#### 问题2

> 我正在做工作总结,下面是我的初步总结，请你帮助我润色下，尽量突出成果，让我的boss能一眼看到我的付出
> 1. 完成了email系统的重构设计和开发，并提前完成了任务，使得iaa, sma, aka等系统的接入更快完成，且支持更高性能以及并发
> 2. 解决了SCV系统遗留的bug,并完成了新业务的开发，提高了系统性能，使得客户使用系统流畅度增加40%。
> 3. 指导初级开发工程师， 帮助他指定完整的学习计划，并帮助他迅速了解业务，快速进入开发角色


![cluade的答案](/images/ai/cluade-answer-2.png)


![deepseek的答案](/images/ai/deepseek-answer-2.png)

这里更加**倾向于claude的答案， 条理更加清晰明确**。

#### 问题3
> 我正在学习mysql的索引优化相关内容，我对索引原理以及mysql内部数据存储结构都比较了解，但是我对索引优化相关知识掌握得并不好,请你给出3个代表性的例子我如何根据explain的结果进行索引优化。 


claude分别给出了我 单表的全表扫描、索引未被充分利用、索引选择性差这三个针对单表的例子，并给出了解决方案

![cluade的答案](/images/ai/cluade-answer-3.png)

deepseek给我了 优化简单查询、优化联合查询、优化排序和分组查询三个案例

![deepseek的答案](/images/ai/deepseek-answer-3.png)

从初步提问来看，deepseek覆盖面更广，而且它还详细给出了表结构语句，我更加**喜欢 deepseek的答案**。

可是给出的sql语句都不太复杂，都需要进一步深入的进行提问

#### 问题4

> 我在做一个需求,我有一个字段，它可能的值如下
>1. (A3 OR A4 OR A5 ) AND ( A1 AND A2 )
>2. A1 AND A2 AND A3
>3. (A9 OR A11 OR A12 ) AND A8 AND A10 AND ( A1 AND A2 AND A3 AND A4 AND A5 AND A6 AND A7)
>4. (A1 AND A2) and (A3 OR A4) and A5 and A6 and (A7 OR A8) and (A9 And A10)
>5. (A9 OR A12 OR A13 ) AND ( A10 OR A11 ) AND A8 AND ( A1 AND A2 AND A3 AND A4 AND A5 AND A6 AND A7 )
>6. A1 AND A2 AND A3 AND A4 AND ( A5 AND A6 AND A7 AND A8 AND A9 AND A10 AND A11)

我需要将其解析成下面的数据结构

```java
public class MultiConditionGroup {

    private String operator;
    private List<ConditionGroup> fieldGroups;

}
public class ConditionGroup {

    private String operator;
    private List<String> fields;
}
``` 

比如对于第一个例子 `(A3 OR A4 OR A5 ) AND ( A1 AND A2 )`, 它最终转换成的MultiConditionGroup的json格式如下

```json
{
  "operator": "AND",
  "fieldGroups": [
    {
      "operator": "OR",
      "fields": [
        "A3", "A4", "A5"
      ]
    }, 
    {
      "operator": "AND",
      "fields": [
        "A1", "A2"
      ]
    }
  ]
}

```

请你根据上面的需求，使用java编写一段转换的代码。

这一段需求是我根据我最近项目的需求转换而来的问题， 我分别问了它们， claude给我哐哐一顿输出代码

![cluade的答案](/images/ai/cluade-answer-4.png)

但是我拿来跑了，首先编译无法通过，然后修改了编译问题，这个case一个都不通过，然后我只能告诉它无法通过单元测试，喊它修改代码，然后它又改了改，还是无法通过，不过这次的代码我可以自己debug下然后改一改就能用了。


然后看看deepseek的答案

![deepseek的答案](/images/ai/deepseek-answer-4.png)

运行deepseek提供的代码，我改了下正则编译问题后直接运行就得到了正确的答案

![deepseek的答案](/images/ai/deepseek-result.png)

并且查看它的思考过程中会发现它把其中两种情况都考虑到了，而我在这之前是没有考虑到的。

比如  `(A1 AND A2) and (A3 OR A4) and A5 and A6 and (A7 OR A8) and (A9 And A10)`

其中的A5 and A6是被当成一组

```json

"fieldGroups": [
    {
      "operator": "AND",
      "fields": [
        "A5", "A6"
      ]
    }
  ]
```

还是A5是一组，A6是一组。 

```json
"fieldGroups": [
    {
      "operator": "AND",
      "fields": [
        "A5"
      ]
    }, 
    {
      "operator": "AND",
      "fields": [
        "A6"
      ]
    }
  ]
```
其实我是认为两种都可以，不过转换的时候我是把它当成一组，然后deepseek认为这存在两种情况，可能在转换的时候需要额外考虑。


从这一组问题看来，虽然deepseek输出相对慢一些(因为它有一个思考的过程)，但是结果更加准确，而且它输出的思考的结果我认为有时候比代码更重要一些，因为从中可以看到它思考的逻辑，就感觉有一个人在和你讨论一样。

这一组问题，**deepseek大比分胜出。**

### 测试结果

我上面测试的问题虽然不多，但是确实是平时都会问到的， 总结类的感觉都大差不差， deepseek略差一些，但是后面需要深度思考的问题deepseek的答案更棒,可以说是惊艳到我了，就感觉有一个人和你讨论，还能给出你建议一样。 

### 服务器繁忙

由于使用的人太多了，在加上存在一些恶意的攻击，使用官方app的时候，总是提示服务器超时，一点都不稳定。
这里我总结了18个可以使用DeepSeek的曲线救国的平替方案。

1. **秘塔搜索**：https://metaso.cn  
2. **360纳米AI搜索**：https://www.n.cn
3. **国家超算互联网**: https://chat.scnet.cn
4. 硅基流动：https://cloud.siliconflow.cn
5. 字节跳动火山引擎：https://console.volcengine.com
6. 百度云千帆：https://console.bce.baidu.com/qianfan
7. 英伟达NIM：https://build.nvidia.com/deepseek-ai
8. Groq：https://groq.com
9. Fireworks：https://fireworks.ai/models/fireworks/deepseek-r1
10. Chutes：https://chutes.ai/app/chute/ 
11. Github(魔法)：https://github.com/marketplace/models/azureml-deepseek/DeepSeek-R1/playground
12. POE(魔法)：https://poe.com/DeepSeek-R1 
13. Cursor(魔法)：https://cursor.sh  
14. Monica(魔法)：https://monica.im
15. Lambda(魔法)：https://lambdalabs.com/  
16. Cerebras(魔法)：https://cerebras.ai
17. Perplexity(魔法)：https://www.perplexity.ai 
18. 阿里云百炼：https://api.together.ai/playground/chat/deepseek-ai/DeepSeek-R1

使用下来我比较推荐前三个，不用登录直接实现，现在可以直接使用。 