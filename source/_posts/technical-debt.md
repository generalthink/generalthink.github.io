---
title: 写过那么多代码,这些问题还在犯吗？？
date: 2021-07-24 17:32:28
tags: [Java]
---

当审视你现在所做的系统，下面这些技术债你遇到了多少？

1. 注释不规范

    类注释，方法注释，语句注释没有一个统一的规范，大多时候没有注释。
   
    建议在关键逻辑上写明注释，不为别人也为自己，几个月后再来看代码，看不懂了就尴尬了。

2. 单元注释少

    建议在核心业务逻辑上添加单元注释
   
3. 大量的sql语句在xml中

    建议少一些，尽量使用面向对象编程(或者面向领域驱动)

4. mybatis-plus queryWrapper使用不正确

    queryWrapper.eq("id", id)类似这种的建议写成 queryWrapper.eq(User::getId, id),避免书写
错误的发生或后续数据库改动导致整个程序都要改。

5. 一个类或者一个方法逻辑太长

    建议一个类300行，一个方法核心代码30行，不要超过阿里的80行。

6. 避免循环嵌套try-catch

7. 分支语句判断不要超过三层，for循环同理

8. 对象锁要使用全局锁，比如private final Object lock = new Object()

9. 能提前判断程序快速结束的，先快速结束

10. 使用StringUtils.equals判断两个字符串是否相等

11. map的迭代使用entrySet方式

12. for循环中建议使用break而不是return

13. 不要混淆使用Boolean和boolean

14. log日志输出，使用占位符而不是字符串拼接

15. List,Map初始化建议使用guava Lists,Maps进行初始化

16. 不用将Map,List等缺乏领域含义的对象用做参数传递

17. 在同一个配置类中进行线程次创建

18. 遵循单一职责原则，将相同的业务处理放置到同一个类中

19. 不要盲目建立索引，而是有的放矢。 索引也不是越多越好

20. mongodb内嵌文档不要太多，不要太大

21. 数据库字段的逻辑删除尽量不要介入业务逻辑

22. 不要用两个字段来表示一个状态

23. 要在关键业务逻辑加上日志，并打印出有效信息，可以帮助排查问题

24. 方法参数不要超过5个，尽量使用对象进行封装

25. apoll配置的值动态更新后，需要确定你使用到它的地方是否能够自动更新上

26. push代码的时候先确保本地build可以通过，同时使用阿里巴巴插件扫描下提交代码是否存在问题

27. 对包进行合理分层

28. dubbo服务对外提供的api，应该只存在一些接定义以及一些参数或者常量，保持最小化

29. 不要过度设计，保持MVP功能优先

30. 该重构就重构，不要等