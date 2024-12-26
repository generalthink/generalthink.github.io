---
title: 使用SpringEvent,我的接口超时了
date: 2024-11-13 10:22:18
tags: [Spring, Event]
---

最近线上的一个接口前端访问竟然超时了，前端设置的超时时间是10s,也就是说这个接口在10s内都还没有返回数据，同事去看了说这个接口就是一个很简单的新增数据接口，不应该导致超时的呀？

我看了下这个接口的逻辑,在结合日志初步定位是因为Spring Event中调用其他服务接口导致的超时。


### 问题追踪

首先看下这个接口调用的主要逻辑


```java

public void saveData(MergeDataParam param) {

	MergeData data = convert(param);

	dataMapper.insert(data);

	applicationContext.publishEvent(new MergeDataEvent(new MergeDataMessage(param.getDataId())));
}

@EventListener
public void notifyOnboardServiceWhenMergeData(MergeDataEvent event) {

	MergeDataMessage msg = (MergeDataMessage)event.getSource();
	//记录日志
	log.info("notify onbaord service when merge data, data id is {}", msg.getDataId());

	NotifyData notifyData = getData(msg.getDataId());
	Response<Result> response = remoteOnboardService.notify(notifyData);

	// 省略其他业务代码

	log.info("notify result is {}", JSON.toJSONString(response));
}
```

确实逻辑比较简单，主要就是保存数据，然后通知下游系统，但是从日志上来看，日志只打印了 `notify onbaord service when merge data, data id is` 这里的内容，然后前端就超时了。

接着我又去看了下游服务的实现，发现确实有一些超时业务逻辑，但是下游服务是别人写的，我们改不了。


**同事还一直以为这个事件是异步的**,我告诉他不是的，如果要想变成异步的需要自己配置。

要想配置成异步的有两种方式,一种是通过@Async注解,另一种方式是在EventMulticaster中指定线程池。



### @Async注解使用


首先我们需要开启异步支持，然后指定一个异步执行器，不然使用的线程池配置是默认的

```java
@Configuration
@EnableAsync
public class AsyncConfig {

    @Bean(name = "asyncExecutor")
    public Executor asyncExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(3);
        executor.setMaxPoolSize(5);
        executor.setQueueCapacity(100);
        executor.setThreadNamePrefix("NotifyAsyncThread-");
        executor.initialize();
        return executor;
    }
}

```


然后在我们事件监听器上加上@Async注解


```java
@Async("asyncExecutor")
@EventListener
public void notifyOnboardServiceWhenMergeData(MergeDataEvent event) {

	MergeDataMessage msg = (MergeDataMessage)event.getSource();
	//记录日志
	log.info("notify onbaord service when merge data, data id is {}", msg.getDataId());

	NotifyData notifyData = getData(msg.getDataId());
	Response<Result> response = remoteOnboardService.notify(notifyData);

	// 省略其他业务代码

	log.info("notify result is {}", JSON.toJSONString(response));
}

```

然后其他代码不用改变，此时我们的监听器就可以异步执行了。需要记住使用这个注解有一些注意事项

1. 异步方法不能返回值。如果需要返回值，可以考虑使用 `Future<T>` 或 `CompletableFuture<T>`。

2. 异步方法中抛出的异常不会被调用者直接捕获。确保适当处理异常，或考虑使用 `AsyncUncaughtExceptionHandler`。

3. 如果在同一个类中调用异步方法，直接调用不会触发异步行为。这是因为Spring的AOP代理机制导致的。

4. 确保你的 MyEvent 类是线程安全的，因为它可能被多个线程同时访问。

如果对@Async默认线程池感兴趣的同学可以通过这几个类进行进一步查看


1. 处理基础类`AsyncExecutionAspectSupport`,定义了默认执行器

```java
 protected Executor getDefaultExecutor(@Nullable BeanFactory beanFactory) {
     // 省略其他代码
    return (Executor)beanFactory.getBean(TaskExecutor.class);
  }

```

2. 异步执行器配置类`TaskExecutionAutoConfiguration`

```java

@Bean
@ConditionalOnMissingBean
public TaskExecutorBuilder taskExecutorBuilder(TaskExecutionProperties properties,
		ObjectProvider<TaskExecutorCustomizer> taskExecutorCustomizers,
		ObjectProvider<TaskDecorator> taskDecorator) {
	TaskExecutionProperties.Pool pool = properties.getPool();
	TaskExecutorBuilder builder = new TaskExecutorBuilder();
	// 省略其他代码
	return builder;
}
	@Lazy
	@Bean(name = { APPLICATION_TASK_EXECUTOR_BEAN_NAME,
			AsyncAnnotationBeanPostProcessor.DEFAULT_TASK_EXECUTOR_BEAN_NAME })
	@ConditionalOnMissingBean(Executor.class)
	public ThreadPoolTaskExecutor applicationTaskExecutor(TaskExecutorBuilder builder) {
		return builder.build();
	}

```

### 配置时间广播器multicaster


首选创建一个配置类

```java
@Configuration
public class AsynchronousSpringEventsConfig {

    @Bean(name = "applicationEventMulticaster")
    public ApplicationEventMulticaster simpleApplicationEventMulticaster() {
        SimpleApplicationEventMulticaster eventMulticaster = new SimpleApplicationEventMulticaster();
        
         ThreadPoolExecutor e = new ThreadPoolExecutor(10, 20, 10, TimeUnit.SECONDS,
                new ArrayBlockingQueue<>(10),
                new ThreadFactoryBuilder().setNameFormat("notify-onboard-thread-%d").build(),
                new ThreadPoolExecutor.AbortPolicy());

        eventMulticaster.setTaskExecutor(e);
        // 设置错误处理器
        eventMulticaster.setErrorHandler(new ErrorHandler() {
            @Override
            public void handleError(Throwable t) {
                log.error("get error when notify onboard service", t);
            }
        });
        return eventMulticaster;
    }
}

```
然后事件监听器的代码不用修改，发布时间方式也不用修改。

使用这种方式有几个注意点

1. 所有有的事件监听器都变为异步执行。如果某些监听器需要同步执行，你可能需要额外的配置或使用不同的多播器。
2. 这里广播器的名字一定要叫applicationEventMulticaster

```java
// APPLICATION_EVENT_MULTICASTER_BEAN_NAME = applicationEventMulticaster
protected void initApplicationEventMulticaster() {
		ConfigurableListableBeanFactory beanFactory = getBeanFactory();
		
		if (beanFactory.containsLocalBean(APPLICATION_EVENT_MULTICASTER_BEAN_NAME)) {
			this.applicationEventMulticaster =
					beanFactory.getBean(APPLICATION_EVENT_MULTICASTER_BEAN_NAME, ApplicationEventMulticaster.class);
		}
		else {
			this.applicationEventMulticaster = new SimpleApplicationEventMulticaster(beanFactory);
			beanFactory.registerSingleton(APPLICATION_EVENT_MULTICASTER_BEAN_NAME, this.applicationEventMulticaster);
		}
	}

```

这里为什么要指定errorHandler呢？ 是因为如果不指定的话，然后执行的业务逻辑又抛出了异常，这个时候异常是会往外抛的，所以我们就会在执行的方法上加上try-catch, 而有了handler,我们就可以统一处理这个异常问题。


```java
	@Override
	public void multicastEvent(final ApplicationEvent event, @Nullable ResolvableType eventType) {
		ResolvableType type = (eventType != null ? eventType : resolveDefaultEventType(event));
		Executor executor = getTaskExecutor();
		for (ApplicationListener<?> listener : getApplicationListeners(event, type)) {
			if (executor != null) {
				executor.execute(() -> invokeListener(listener, event));
			}
			else {
				invokeListener(listener, event);
			}
		}
	}

	protected void invokeListener(ApplicationListener<?> listener, ApplicationEvent event) {
		ErrorHandler errorHandler = getErrorHandler();
		if (errorHandler != null) {
			try {
				doInvokeListener(listener, event);
			}
			catch (Throwable err) {
				errorHandler.handleError(err);
			}
		}
		else {
			doInvokeListener(listener, event);
		}
	}
```

从 `SimpleApplicationEventMulticaster`中可以看出来，如果不指定Executor, 则最终事件的执行是由同一
个线程按顺序来完成的，同时任何一个报错，都会导致后续的监听器执行不了(如果存在多个监听器)。


### 总结

至此，我总结了两种事件异步执行的方式，我让我同事选择一种，他选了第二种，如果是你你会选择哪一种？