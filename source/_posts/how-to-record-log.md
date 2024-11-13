---
title: 工作8年,你就是这样记日志的？
date: 2024-09-19 17:05:40
tags: [log, traceId, logback, slf4j]
---

平常在写代码的过程中,我们经常需要记录日志,具体的日志实现我们可以使用logback, log4j, log4j2等。 但是一般我们会通过日志门面来记录日志，比如通过SLF4J或者apache commons logging。 无论使用哪种方式记录日志都不在我们这篇文章的讨论范围内。

如果你对它们如何找到真正的日志实现感兴趣，可以看看我[之前的文章](https://juejin.cn/post/6844903936780926989)。

### 日志怎么记录

日志分为6种，每种级别实际上有不同的应对场景，但是说实话我在项目中看到最多的永远是INFO, ERROR级别。

几乎没有怎么见到业务系统中有其他级别。 DEBUG的日志在集成的各个框架中是比较多见的。

日志应该是帮助我们定位问题的，所以日志怎么记录其实很关键。

<!--more-->


### DEBUG日志

#### 开源框架如何使用

哪里应该记录debug日志呢？ 我们先看看开源框架中哪里地方会打印DEBUG日志，我们学习下。

项目中使用了Nacos,在加载nacos配置的变量的时候，会把加载到的Nacos的数据打印出来

```java
// com.alibaba.cloud.nacos.client.NacosPropertySourceBuilder#loadNacosData#92
if (log.isDebugEnabled()) {
	log.debug(String.format(
			"Loading nacos data, dataId: '%s', group: '%s', data: %s", dataId,
			group, data));
}

```
可以看到这里使用的是debug日志。  一般而言我们的配置基本上都是配置到nacos中的，比如数据库连接信息，openfeign配置信息，还有其他一些变量，所以这里的数据打印出来应该是比较多的。 像这种日志打印出来也更多只是为了验证下我们的配置是否正确或者帮助我们调试。

所以这里我们使用debug级别日志来打印信息。

再来看看Spring中哪些地方用到了debug日志。

```java
// org.springframework.beans.factory.support.DefaultListableBeanFactory#registerBeanDefinition#1014
public void registerBeanDefinition(String beanName, BeanDefinition beanDefinition)
			throws BeanDefinitionStoreException {

    // 省略其他代码

	else if (!beanDefinition.equals(existingDefinition)) {
		if (logger.isDebugEnabled()) {
			logger.debug("Overriding bean definition for bean '" + beanName +
					"' with a different definition: replacing [" + existingDefinition +
					"] with [" + beanDefinition + "]");
		}
	}
}

```
当我们注册BeanDefinition的时候，如果有相同的,这时候会把旧的替换掉，所以这里会打上一个debug日志。 当想要排查之前的Bean是怎样被替换的，这时候只需要开启日志级别是DEBUG，我们就能清晰的知道Bean是被那个Defeinition替换掉的了。

这里为什么不记录INFO日志呢？ 我个人理解是因为注册BeanDefinition这个行为如果产生了replace行为,那么一定是使用者控制的，框架不会产生重复的BeanDefinition, 那么既然是使用者自己行为,所以这里替换掉原有的就很合理，但是又为了让用户知道框架的具体行为(是替换还是忽略),所以这里加了一个debug日志。

#### 业务系统如何记录

在业务开发中我们主要是和参数打各种交道,比如我们接收到前端一个大参数的时候,我们需要对这个参数进行各种转换和计算之后保存到数据库。这个时候我建议记录下DEBUG日志。 

有时候业务逻辑依赖这个参数进行运算，但是有时候得到的结果又和我们需要的不一致，所以这时候就需要排查下前端是否处理的不准确，或者计算逻辑是否不正确。 这个时候把参数打印出来我们就可以复现这个问题。 

```java
log.debug("save param is {}", JSON.toJSONString(param));
```
这里打印的时候我建议打印出json字符串,因为我们拿到这个字符串后可以在本地进行调试。当然由于这个字符串很大,这也是我们设置成debug级别的原因之一,至少不要让日志成为制约系统响应速度的罪魁祸首。


### INFO日志

#### 开源框架如何使用

同样是nacos,在进行服务注册的时候如果注册成功，会输出一条注册完成的日志。从这个日志我们可以知道哪个服务注册成功了。

```java
// com.alibaba.cloud.nacos.registry.NacosServiceRegistry:register#75
log.info("nacos registry, {} {} {}:{} register finished", group, serviceId,
					instance.getIp(), instance.getPort());

```
从这里可以看到使用INFO日志,是记录一个比较重要的时刻,我们打印出必要的消息，证明已经跑过了一个重要的节点。


同样的在Spring初始化ServletContext的时候也打印了INFO日志

```java
// org.springframework.boot.web.servlet.context.ServletWebServerApplicationContext#prepareWebApplicationContext#288
setServletContext(servletContext);
			if (logger.isInfoEnabled()) {
				long elapsedTime = System.currentTimeMillis() - getStartupDate();
				logger.info("Root WebApplicationContext: initialization completed in " + elapsedTime + " ms");
			}

```
从这里我们可以看出来测试Spring框架运行到了一个重要的节点,又打印出了一些必要的信息：初始化的时间。

#### 业务系统如何记录

我们一般会在一些重要的节点进行日志记录,比如在邮件模板系统中,当我们找到需要处理的模板的时候,我记录了这样的一条日志

```java
log.info("find templates by rule,templateNames={}, serviceLineId={}, companyId={}, triggerEventId={}",
                emailTemplateList.stream().map(EmailTemplate::getTemplateName).collect(Collectors.joining(";", "[", "]")),
                dto.getServiceLineId(), dto.getCompanyId(), dto.getTriggerEventId()
        );

```
这条日志记录了我根据参数找到了哪些需要被处理的模板。后面当我看日志的时候我就知道根据下游系统的参数定位到了哪些模板来处理

### WARN日志

#### 开源框架如何使用

同样的，在Nacos中如果未加载到nacos的配置信息的时候，这里使用了warn日志

```java
// com.alibaba.cloud.nacos.client.NacosPropertySourceBuilder#loadNacosData#86
if (StringUtils.isEmpty(data)) {
	log.warn(
			"Ignore the empty nacos configuration and get it based on dataId[{}] & group[{}]",
			dataId, group);
	return Collections.emptyList();
}
```
这里为什么不使用ERROR其他类型的日志呢？ 其实是因为加载不到配置信息不影响系统的运行,但是你用了nacos又加载不到，就很奇怪。

所以这里使用了warn级别日志，而不是ERROR或者其他级别日志。 

#### 业务系统如何记录

warn是一个警告，证明了这个信息不影响业务继续运行,但是会对业务使用有一定影响。比如我们列表查询需要展示业务单独的操作人,根据单据存储的userId获取userName进行展示, 需要去用户服务中取,但是由于下游系统挂了或者这个用户被删除了， 我们取不到这个用户了。

那整个列表就不展示了吗？ 当然不行,顶多是某行数据的user name不展示就行了。 这种情况下,我们需要记录下warn日志。

```java
public String getUserById(Long id) {
	// 省略业务代码

	if(user == null) {
		log.warn("user not exists, userId={}", id);
		return null;
	}
}

```

### ERROR日志

#### 开源框架如何使用

同样还是nacos加载数据失败的场景，这里就打印了通过哪个dataId加载数据的时候失败，并打印了异常。
但是因为nacos加载数据失败并不会影响程序的继续运行，所以这里这是打印了日志，并没有抛出异常。

```java
private List<PropertySource<?>> loadNacosData(String dataId, String group,
		String fileExtension) {
	catch (NacosException e) {
		log.error("get data from Nacos error,dataId:{} ", dataId, e);
	}
	catch (Exception e) {
		log.error("parse data from Nacos error,dataId:{},data:{}", dataId, data, e);
	}
	return Collections.emptyList();
}

```

而在Spring初始化ServletContext的时候如果失败了是会打印error日志，并抛出异常的。

```java

protected void prepareWebApplicationContext(ServletContext servletContext) {
	try {
			
		setServletContext(servletContext);
		// 省略其他代码
	}
	catch (RuntimeException | Error ex) {
		logger.error("Context initialization failed", ex);
		servletContext.setAttribute(WebApplicationContext.ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE, ex);
		throw ex;
	}

}

```
这里不仅仅记录了错误日志，而且还抛出了异常，而这个异常会直接让我们的应用程序崩溃，这很合理，因为servlet初始化失败，就代表我们的容器启动失败，所以这里就需要住址程序继续往下运行。


#### 业务系统如何记录

业务系统中如果我们捕获了某些异常，然后需要记录错误日志，比如我们发送测试邮件失败。但是又因为发送邮件不是一个必须行为，所以我们并不抛出异常，而是可以继续执行后续逻辑。

```java
 try {
        remoteNotiService.sendContent(email);
    } catch (Exception e) {
        log.error(LogUtil.format("send email failure, toList={}, subject={}", toList, subject), e);
    }

```

当然如果你在系统中捕获的非业务异常,那么一般都是记录日志，然后抛出异常的。

### 添加traceId



最后无论现在你们是否有完整的日志体系，我都建议你把traceId加入到你的日志中,因为你可以通过traceId拉到某一次请求所有的日志，能很好的帮助你过滤掉无关日志。 关于traceId的添加可以参考我[之前的文章](https://juejin.cn/post/6949073119852101663)

虽然有了traceId,但是我还是建议你记录日志的文本要是唯一的,这样你根据日志搜索的时候可以直接定位到是哪一段代码。

当然我这里还推荐给你了另外一种方式,如果你使用了spring cloud, 那你可以使用sleuth来添加traceId。 


```xml
 <dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-sleuth</artifactId>
</dependency>
```
此时你只需要在你spring cloud的gateway中添加一个filter即可。

```java
@Component
public class TraceIdFilter  implements GlobalFilter, Ordered {
    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        ServerHttpResponse response = exchange.getResponse();
        HttpHeaders httpHeaders = response.getHeaders();
        httpHeaders.add("X-Trace-Id", MDC.get("traceId"));
        return chain.filter(exchange.mutate().response(response).build());
    }

    @Override
    public int getOrder() {
        return -999;
    }
}
```
注意这里是从MDC(slf4j)中获取的，因为sleuth会将这个值放到MDC中，我们直接从里面获取就行了。

当然你可以直接注入`org.springframework.cloud.sleuth.Tracer`,然后这样获取traceId也可以使用的

```java
tracer.currentSpan().context().traceId();
```

然后在logback的配置文件的log pattern中加上traceId,这里我给出我的log pattern

```xml
 <property name="log.pattern" value="%d{HH:mm:ss.SSS}-[%thread]-[traceId=%X{traceId},spanId=%X{spanId}]-%-5level-%logger{20}-[%method,%line]-%msg%n" />
```

这样子打印出来的日志就会有traceId了，然后返回给前端的header里面也带有traceId了。而且因为引入了sleuth,所以它只天然支持openfeign的，你就不用再去写Inteceptor了。


### 总结

我上面特意忽略了trace和fatal这两个级别，这两个实在不常用,不过你可以根据你的实际情况来使用，这里我也把各个级别的使用场景总结了下。

#### TRACE

+ 用于记录程序执行的细节信息,提供最详细的日志信息。
+ 通常用于调试和问题分析,在正常运行时可以被禁用以减少日志输出。

#### DEBUG

+ 用于记录程序执行的详细信息,有助于开发和调试。
+ 通常在开发和测试环境中使用,在生产环境中可以被禁用以减少日志输出。

#### INFO

+ 用于记录程序执行的重要信息,如关键事件、状态变更等。
+ 通常在生产环境中使用,提供程序运行的概况信息。

#### WARN

+ 用于记录程序执行中出现的潜在问题,但不会导致程序崩溃。
+ 通常用于记录一些可能会影响程序正常运行的情况,如资源耗尽、配置错误等。

#### ERROR

+ 用于记录程序执行中出现的错误信息,可能会导致程序崩溃或无法正常运行。
+ 通常用于记录严重的问题,如异常、错误代码、系统故障等。


#### FATAL

+ 用于记录程序执行中出现的严重错误,导致程序无法继续运行。
+ 通常用于记录程序无法恢复的致命错误,如系统崩溃、资源耗尽等。

