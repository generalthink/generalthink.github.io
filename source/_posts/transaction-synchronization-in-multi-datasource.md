---
title: 不同数据源的事务同步
date: 2024-11-13 16:24:45
tags: [Spring, Transaction, jta]
---

### 现象

最近线上的一条数据状态不对，但是日志又记录上了。 查看了这条数据的更新逻辑

```java
  public Boolean autoReject(AutoRejectParam param) {
        
        OperationLog log = createOperationLog(param);

        // 保存操作日志到mysql
 		operationLogMapper.insertSelective(log);

        Query query = new Query();
        Criteria criteria = new Criteria();
        criteria.and("requestId").is(param.getRequestId());
        query.addCriteria(criteria);
        Update update = new Update();
        update.set("status", CvBusinessStatusEnum.Rejected.getCode())
                .set("updateTime", new Date())
                .set("taskId", "");
        mongoTemplate.updateFirst(query, update, JSONObject.class, collectionName);

        return true;
    }

```

从代码可以看出这里分别保存了日志到mysql,然后更新了mongodb中的数据状态。


很明显保存mysql成功了,但是更新mongodb的数据失败了,那为什么保存mongodb的数据失败了呢？ 然后根据日志发现,当时服务器和mongodb连接出现了问题,于是就导致了保存mysql成功，保存到mongodb失败了。


### 如何解决？

问题既然产生了，那么有什么办法能够保证要成功就都成功呢？ 第一个想到的是事务,我们需要保证两个数据库操作的事务一致性就可以避免这个问题了。使用单一的事务管理器肯定是不行的，需要使用链式事务。

我们可以使用spring中的`ChainedTransactionManager`来实现链式调用

```java
@Configuration
public class TransactionConfig {

    @Bean
    public PlatformTransactionManager mongoTransactionManager(MongoTemplate mongoTemplate) {
        return new MongoTransactionManager(mongoTemplate.getMongoDbFactory());
    }

    @Bean
    public PlatformTransactionManager jpaTransactionManager(DataSource dataSource) {
        return new DataSourceTransactionManager(dataSource);
    }

    @Bean
    public ChainedTransactionManager chainedTransactionManager(
            PlatformTransactionManager mongoTransactionManager,
            PlatformTransactionManager jpaTransactionManager) {
        return new ChainedTransactionManager(mongoTransactionManager, jpaTransactionManager);
    }
}


@Transactional("chainedTransactionManager")
public Boolean autoReject(AutoRejectParam param) {
        
       	//省略其他代码 

        // 保存操作日志到mysql
 		operationLogMapper.insertSelective(log);

 		// 更新mongodb
        mongoTemplate.updateFirst(query, update, JSONObject.class, collectionName);

        return true;
  }

```
这种方法使用 ChainedTransactionManager 来管理多个事务管理器。当方法执行时，它会按顺序开启所有事务，如果在执行过程中出现异常，它会按相反的顺序回滚所有事务。

需要注意的是，这种方法并不能保证 100% 的事务一致性，因为它实际上是在应用层面模拟的分布式事务。在某些极端情况下（比如网络故障或服务器崩溃），可能会出现部分提交的情况。


比如我们是现在这样的执行流程

```
transaction1 begin
  transaction2 begin
  transaction2 commit -> error rollbacks, rollbacks transction1 too
transaction1 commit -> error, only rollbacks transaction1

```

比如上面这种情况,在最后提交transaction1的时候如果由于网络原因提交失败了，就会导致事务2成功，事务1失败，还是部分提交了。

当然如果业务要求对于这种不一致是可以接受的，或者说我们可以进行手动补偿方式达到最终一致性，那这种方案也是可以接收的。

对于要求更高事务一致性的场景，可能需要考虑使用专门的分布式事务解决方案，如 XA 协议或 TCC (Try-Confirm-Cancel) 模式。
比如JTA就属于XA协议， 我们可以使用开源实现atomikos。