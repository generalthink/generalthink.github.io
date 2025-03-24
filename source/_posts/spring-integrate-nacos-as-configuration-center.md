---
title: springcloud是如何集成nacos作为配置中心的
date: 2025-03-01 17:29:24
tags: [SpringCloud, Nacos]
---

你的SpringCloud项目中使用了Nacos作为配置中心，你是否好气它是如何和SpringCloud进行集成的， 那么我们就走进源码来看看,要分析如何集成的，我们首先要引入对应的依赖

### 引入依赖

```
<dependency>
    <groupId>com.alibaba.cloud</groupId>
    <artifactId>spring-cloud-starter-alibaba-nacos-discovery</artifactId>
    <version>2021.1</version>
</dependency>

<dependency>
    <groupId>com.alibaba.cloud</groupId>
    <artifactId>spring-cloud-starter-alibaba-nacos-config</artifactId>
    <version>2021.1</version>
</dependency>

```
### 配置

```
server:
  port: 9204

# Spring
spring: 
  application:
    # 应用名称
    name: think123-cert
  profiles:
    # 环境配置
    active: ${activeProfile}
  cloud:
    nacos:
      discovery:
        # 服务注册地址
        server-addr: 127.0.0.1:8848
#       namespace: 223db9d4-e3ab-4e61-9293-3f1da8dd8448
      config:
        # 配置中心地址
        server-addr: 127.0.0.1:8848
        # 配置文件格式
        file-extension: yml
        # 共享配置
        shared-configs:
          - application-${spring.profiles.active}.yml

```



#### 源码探索


#### 加载入口： spring.factories

我们可以从spring-cloud-starter-alibaba-nacos-config中的spring.factories中发现启动的时候会加载两个Configuration

```
org.springframework.cloud.bootstrap.BootstrapConfiguration=\
com.alibaba.cloud.nacos.NacosConfigBootstrapConfiguration
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
com.alibaba.cloud.nacos.NacosConfigAutoConfiguration,\
```

但是SpringCloud的加载机制会早于SpringBoot的加载机制,所以这里是`NacosConfigBootstrapConfiguration`先执行

#### 配置拉取

NacosConfigBootstrapConfiguration这里面首先回去初始化一个NacosConfigManager
```java
	

	@Bean
	@ConditionalOnMissingBean
	public NacosConfigProperties nacosConfigProperties() {
		// 处理spring.cloud.nacos.config配置
		return new NacosConfigProperties();
	}

	@Bean
	@ConditionalOnMissingBean
	public NacosConfigManager nacosConfigManager(
			NacosConfigProperties nacosConfigProperties) {
		// 这里会创建NacosConfigService
		return new NacosConfigManager(nacosConfigProperties);
	}

	@Bean
	public NacosPropertySourceLocator nacosPropertySourceLocator(
			NacosConfigManager nacosConfigManager) {
		// 拉取配置
		return new NacosPropertySourceLocator(nacosConfigManager);
	}
```
而NacosConfigService会启动两个线程(通过线程池启动的), 其中ServierListManager会创建GetServerListTask去拉取服务器列表，然后这个定时任务是每隔30s运行一次。

```java
 public NacosConfigService(Properties properties) throws NacosException {
        ValidatorUtils.checkInitParam(properties);
        
        initNamespace(properties);
        this.configFilterChainManager = new ConfigFilterChainManager(properties);
        ServerListManager serverListManager = new ServerListManager(properties);
        // 拉取服务器列表
        serverListManager.start();
        
        this.worker = new ClientWorker(this.configFilterChainManager, serverListManager, properties);
        // will be deleted in 2.0 later versions
        agent = new ServerHttpAgent(serverListManager);
        
  }

   // com.alibaba.nacos.client.config.impl.ServerListManager
   public synchronized void start() throws NacosException {
        
        if (isStarted || isFixed) {
            return;
        }
        
        GetServerListTask getServersTask = new GetServerListTask(addressServerUrl);
        for (int i = 0; i < initServerlistRetryTimes && serverUrls.isEmpty(); ++i) {
            getServersTask.run();
            //省略部分代码
        }
        
        // ScheduledThreadPoolExecutor，每隔30秒拉取一次服务器列表
        this.executorService.scheduleWithFixedDelay(getServersTask, 0L, 30L, TimeUnit.SECONDS);
        isStarted = true;
    }

```


分别获取服务器列表以及启动一个ClientWorker，它的主要作用是同步和处理缓存配置的监听，你可以认为就是它在比较你本地运行的配置和服务器的配置是否一致，如果不一致就通知监听器，这时候监听器就会去处理,此时你的配置值就已经是最新的了。


```java


private final BlockingQueue<Object> listenExecutebell = new ArrayBlockingQueue<Object>(1);

// com.alibaba.nacos.client.config.impl.ClientWorker
@Override
public void startInternal() throws NacosException {
    executor.schedule(() -> {
        while (!executor.isShutdown() && !executor.isTerminated()) {
            try {
            	// 每隔5s监听
                listenExecutebell.poll(5L, TimeUnit.SECONDS);
                if (executor.isShutdown() || executor.isTerminated()) {
                    continue;
                }
                executeConfigListen();
            } catch (Exception e) {
                LOGGER.error("[ rpc listen execute ] [rpc listen] exception", e);
            }
        }
    }, 0L, TimeUnit.MILLISECONDS);
    
}

```

之前初始化只加载了服务器列表,初始化了NacosConfigService, 启动了线程去监听配置是否有变化，但是没看到哪里去拉取配置呢？

这里就需要我们的NacosPropertySourceLocator,它会在启动的时候就去加载我们的配置

```java

	// 通过 PropertySourceBootstrapConfiguration被调用
	@Override
	public PropertySource<?> locate(Environment env) {
		nacosConfigProperties.setEnvironment(env);
		ConfigService configService = nacosConfigManager.getConfigService();

		if (null == configService) {
			log.warn("no instance of config service found, can't load config from nacos");
			return null;
		}
		long timeout = nacosConfigProperties.getTimeout();
		nacosPropertySourceBuilder = new NacosPropertySourceBuilder(configService,
				timeout);
		String name = nacosConfigProperties.getName();

		String dataIdPrefix = nacosConfigProperties.getPrefix();
		if (StringUtils.isEmpty(dataIdPrefix)) {
			dataIdPrefix = name;
		}

		if (StringUtils.isEmpty(dataIdPrefix)) {
			dataIdPrefix = env.getProperty("spring.application.name");
		}

		CompositePropertySource composite = new CompositePropertySource(
				NACOS_PROPERTY_SOURCE_NAME);

		loadSharedConfiguration(composite);
		loadExtConfiguration(composite);

		loadApplicationConfiguration(composite, dataIdPrefix, nacosConfigProperties, env);
		return composite;
	}

	private void loadApplicationConfiguration(
			CompositePropertySource compositePropertySource, String dataIdPrefix,
			NacosConfigProperties properties, Environment environment) {

		...

		loadNacosDataIfPresent(compositePropertySource, dataIdPrefix, nacosGroup,
				fileExtension, true);
		...
	}
```

最终会调用我们之前说过的NacosConfigService::getConfig方法从服务器端拉取配置,最后通过PropertySourceLoader解析成PropertySource对象，这样Spring就可以进行统一处理了。

#### 配置监听

再来看看我们的NacosConfigAutoConfiguration

```java


@Bean
public NacosRefreshHistory nacosRefreshHistory() {
	return new NacosRefreshHistory();
}
@Bean
public NacosContextRefresher nacosContextRefresher(
		NacosConfigManager nacosConfigManager,
		NacosRefreshHistory nacosRefreshHistory) {
	// Consider that it is not necessary to be compatible with the previous
	// configuration
	// and use the new configuration if necessary.
	return new NacosContextRefresher(nacosConfigManager, nacosRefreshHistory);
}
```

`NacosRefreshHistory`是用于记录当前Nacos的配置刷新了多少次的，最多只记录20次。而NacosContextRefresher则是用于
注册监听器

```java
private void registerNacosListener(final String groupKey, final String dataKey) {
		String key = NacosPropertySourceRepository.getMapKey(dataKey, groupKey);
		Listener listener = listenerMap.computeIfAbsent(key,
				lst -> new AbstractSharedListener() {
					@Override
					public void innerReceive(String dataId, String group,
							String configInfo) {
						refreshCountIncrement();
						//记录更新历史
						nacosRefreshHistory.addRefreshRecord(dataId, group, configInfo);
						// 发布属性更新事件
						applicationContext.publishEvent(
								new RefreshEvent(this, null, "Refresh Nacos config"));
						if (log.isDebugEnabled()) {
							log.debug(String.format(
									"Refresh Nacos config group=%s,dataId=%s,configInfo=%s",
									group, dataId, configInfo));
						}
					}
				});
		// 注册监听器
		configService.addListener(dataKey, groupKey, listener);
		
	}
```

这里注册监听器会将CacheData写入到一个缓存中,而ClientWorker就会去比较它们的MD5值,如果有变化就通过RefreshEvent来更新整个Spring Context中的属性值。

如果你想看它的详细流程，你还可以通过日志进行辅助

```yaml
logging:
  level:
    com.alibaba.nacos: DEBUG
    com.alibaba.cloud.nacos: DEBUG
```

### 总结

通过上面的源码，我们知道了SpringCloud是如何集成了Nacos作为配置中心的,我们也知道你在服务器修改的配置最多5秒就会被你的应用程序感知到，从而生效。最后我在来给你总结阅读源码关键点

#### 自动装配
- **入口类**：`META-INF/spring.factories`
- **Config管理**：`NacosConfigBootstrapConfiguration` → 创建`NacosConfigManager`从而创建`NacosConfigService`
- **拉取配置**：`NacosConfigBootstrapConfiguration` → 创建`NacosPropertySourceLocator`
- **NacosConfigAutoConfiguration** → 配置刷新器`NacosContextRefresher`


#### 配置加载流程
1. `NacosPropertySourceLocator` 从远程拉取配置
2. `NacosContextRefresher` 监听配置变更事件
3. `RefreshEvent` 触发Bean属性刷新

*关键类：`NacosConfigManager`（配置管家）*

*有趣发现：使用`ScheduledThreadPoolExecutor`作为线程池*

---

#### 日志秘籍
```yaml
logging:
  level:
    com.alibaba.nacos: DEBUG
    com.alibaba.cloud.nacos: DEBUG
```

