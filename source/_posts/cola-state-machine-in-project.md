---
title: 我在项目中这样使用状态机
date: 2023-11-16 15:30:15
tags: [cola, statemachine,状态机]
---

最近一个新的项目中的一个业务,状态的流转比较复杂，涉及到二十几个状态的流转,而且吸取了其他业务教训,我们决定使用状态机来解决状态流转的问题。


要使用状态机除了自己写状态模式下还研究了当下两个开源项目，一个是spring的state machine,一个是cola-state-machine。


spring的状态机可以做状态持久化,和spring结合比较好，但是太重了。 cola就比较简单,它只是简单做了一个抽象,我们只需要实现具体的行为就行了。 使用cola最重要的就是要记得"因为某个事件，导致了状态A向状态B进行了迁移",当然这里的状态可以是同一个。


因为项目中使用的是springboot,所以我这里结合起来做了一定的改造,下面给出我在项目中使用的例子,仅供大家参考


### 引入依赖

```xml
<dependency>
	<groupId>com.alibaba.cola</groupId>
	<artifactId>cola-component-statemachine</artifactId>
	<version>4.3.2</version>
</dependency>

```

### 定义

因为某个事件，导致了状态A向状态B进行了迁移。所以需要定义状态,事件,流程。

![状态迁移](/images/state-machine/state-process.png)

#### 事件

根据我们的流程我定义了以下事件

```java
@AllArgsConstructor
@Getter
public enum StatusChangeEventEnum {
	// save as draft
	SAVE_AS_DRAFT_EVENT,
	// draft submit
	DRAFT_SUBMIT_EVENT,
	// submit
	SUBMIT_EVENT;
}

```
#### 状态

```java
@Getter
@AllArgsConstructor
public enum StatusEnum {

	NONE(0, "None"),
	DRAFT(1, "Draft"),
	SUBMITTED(2, "Submitted");

 	private Integer code;
    private String desc;
}

```

#### 流程

定义状态迁移和事件的关系

```java
@Getter
@AllArgsConstructor
public enum StatusChangeEnum implements StatusChange {

	SAVE_AS_DRAFT(NONE, DRAFT, SAVE_AS_DRAFT_EVENT),
	SUBMIT(NONE, SUBMITTED, SUBMIT_EVENT),
	DRAFT_SUBMIT(DRAFT, SUBMITTED, DRAFT_SUBMIT_EVENT);

	private StatusEnum fromStatus;
    private StatusEnum toStatus;
    private StatusChangeEventEnum event;

 	@Override
    public StatusEnum from() {
        return fromStatus;
    }

    @Override
    public StatusEnum to() {
        return toStatus;
    }

    @Override
    public StatusChangeEventEnum event() {
        return event;
    }

}

// 抽象状态变更的接口，因为可能会存在多个不同的状态变更流程
public interface StatusChange {

    StatusEnum from();

    StatusEnum to();

    StatusChangeEventEnum event();

}

```

### 使用Spring管理状态机

```java

@Configuration
@RequiredArgsConstructor
public class StateMachineConfiguration {
	private final List<StateMachineHandler> handlerList;
	private static final String NEW_REQUEST_STATE_MACHINE = "newRequestStateMachine";

	@Bean(NEW_REQUEST_STATE_MACHINE)
	public <C> StateMachine<StatusEnum, StatusChangeEventEnum, C> newRequestStateMachine() {

	    StateMachineBuilder<StatusEnum, StatusChangeEventEnum, C> builder = StateMachineBuilderFactory.create();

	    for (StatusChangeEnum changeEnum :StatusChangeEnum.values()) {
	        build(builder, changeEnum);
	    }
	    return builder.build(NEW_REQUEST_STATE_MACHINE);
	}

	private <C> void build(StateMachineBuilder<StatusEnum, StatusChangeEventEnum, C> builder, StatusChange statusChange) {
		// 找到对应的handler来处理
	    StateMachineHandler<StatusEnum, StatusChangeEventEnum, C> handler = getHandler(statusChange);

	    StatusEnum fromStatus = statusChange.from();
	    StatusEnum toStatus = statusChange.to();
	    StatusChangeEventEnum changeEvent = statusChange.event();
	    // 只产生了事件,但是状态未发生变化
	    if (fromStatus == toStatus) {
	        builder.internalTransition()
	                .within(fromStatus)
	                .on(changeEvent)
	                .when(handler::isSatisfied)
	                .perform((from, to, event, ctx) -> handler.execute(from, to, event, ctx));
	    } else {
	        builder.externalTransition()
	                .from(fromStatus)
	                .to(toStatus)
	                .on(changeEvent)
	                .when(handler::isSatisfied)
	                .perform((from, to, event, ctx) -> handler.execute(from, to, event, ctx));
	    }
	    // 直接在handler中抛出更详细异常
	    //builder.setFailCallback(new AlertFailCallback<>());
	}

	private <C> StateMachineHandler<StatusEnum, StatusChangeEventEnum, C> getHandler(StatusChange statusChange) {
	    return handlerList.stream().filter(handler -> handler.canHandle(statusChange))
			    .findFirst()
			    .orElse(new DefaultStateMachineHandler<C>());
	}

}

```

经过上面的定义后,后续有新的状态变更流程,我们只需要在 `StatusChangeEnum` 中添加就行了。


### 实现对应的handler

这里我举一个例子,比如说首次提交数据

```java

@Component
@RequiredArgsConstructor
public class NoneToSubmittedStatusHandler  implements 
						StateMachineHandler<StatusEnum, StatusChangeEventEnum, SubmitDTO> {
    @Override
    public boolean canHandle(StatusChange statusChange) {
    	// handler能处理的变更流程
        StatusChangeEnum changeEnum = StatusChangeEnum.SUBMIT;
        return statusChange == changeEnum;
    }

    @Override
    public void execute(StatusEnum from, StatusEnum to, StatusChangeEventEnum event, SubmitDTO context) {
    	// 执行具体的业务逻辑,比如插入数据
    }

    @Override
    public boolean isSatisfied(SubmitDTO context) {
        // 判断是否满足条件,比如是否是对应的用户等
        return true;
    }
}

```
这样子我们的每一个handler的功能就比较专一了,只需要处理对应状态的就行了，你可能回想要是有些状态的变成要做的事情类似,这样的代码不可能写两遍吧? 其实我们可以有一个抽象类可以将这些公用的逻辑放到抽象类里面,这样子有相同逻辑的就可以使用了。


### 使用

万事具备,现在只差在项目中使用了

```java
@Component
@RequiredArgsConstructor
public class StateMachine {

 private final StateMachine<StatusEnum, StatusChangeEventEnum, StatusChangeContext> newRequestStateMachine;


  	@Transactional(rollbackFor = Exception.class)
    public void statusChange(StatusChange changeEnum, StatusChangeContext context) {
        newRequestStateMachine.fireEvent(changeEnum.from(), changeEnum.event(), context);
    }

}

public abstract class StatusChangeContext {
    
}

@Data
public class SubmitDTO extends StatusChangeContext {

	private Long id;

	private Long String;
}

```

只要涉及到状态变更的,就都可以调用StateMachie了。 


### 写到最后

这种方式其实会导致类的数量变多，但是职责更加清晰,每个类的代码行数也并不多，而且以后想要找某个状态变更到某个状态做了什么时候很很好找。

这就是最近使用状态机的一些心得，希望能对你有所帮助。