---
title: 因为一个bug,我还是掀开了openfeign的神秘面纱
date: 2023-12-14 10:41:39
tags: [SpringCloud, OpenFeign, Feign]
---

### 报错

最近项目中访问一个外部api报错了，报错信息如下

```
PKIX path building failed: sun.security.provider.certpath.SunCertPathBuilderException: unable to find valid certification path to requested target
```
看着像是证书问题，这个时候我首先想到的是百度下，看看怎么解决。

### 解决方案

百度告诉我说如果你open-feign中使用的是http client,那么可以通过下面的配置来让跳过SSL验证

```yml
feign:
  httpclient:
    disable-ssl-validation: false
```

结果还是报同样的错误。 于是我又百度,又重新找了一个解决方法，这次的方案是让我自己重写Client了,具体操作如下

```java
@Configuration
public class FeignConfiguration {

@Bean
public Client feignClient() throws NoSuchAlgorithmException, KeyManagementException {
    SSLContext ctx = SSLContext.getInstance("SSL");
    X509TrustManager tm = new X509TrustManager() {
        @Override
        public void checkClientTrusted(X509Certificate[] chain, String authType) {
        }
        @Override
        public void checkServerTrusted(X509Certificate[] chain, String authType) {
        }
        @Override
        public X509Certificate[] getAcceptedIssuers() {
            return null;
        }
    };
    ctx.init(null, new TrustManager[]{tm}, null);


    return  new Client.Default(ctx.getSocketFactory(), (hostName, session) -> true);

}
   
}

```
这把我感觉要起飞了， 一切尽在掌握中,重新deploy,打开postman,测试测试我的接口。

测试后感觉好了但是看日志又没有完全好。 这个接口倒是不报错了，但是我调用内部服务给我报错了，比如我这里的内部服务名称叫做

pro-file, 就现在它没法根据我这个pro-file名字找到对应的IP了，从而导致我这个服务使用不了了。

百度误我！

### 求人不如求己


此刻我自信的打开了IDEA, 输入了类名 `FeignAutoConfiguration` , Spring Cloud关于某个组件的自动注入类大多是XXXConfiguration, 所以按照这么找准没错。

然后我有自信的把断电打在了这个部分 `FeignAutoConfiguration:246`

```java
	@Configuration(proxyBeanMethods = false)
	@ConditionalOnClass(ApacheHttpClient.class)
	@ConditionalOnMissingBean(CloseableHttpClient.class)
	@ConditionalOnProperty(value = "feign.httpclient.enabled", matchIfMissing = true)
	@Conditional(HttpClient5DisabledConditions.class)
	protected static class HttpClientFeignConfiguration {
		// 省略其他代码

		@Bean
		@ConditionalOnMissingBean(Client.class)
		public Client feignClient(HttpClient httpClient) {
			return new ApacheHttpClient(httpClient);
		}

}
```
重新启动项目，好家伙断点没进来呀。 没进来的原因大概率可能是不满足条件,我赶紧看看这里对应的Conditional, 发现了我的代码中没有

设置`feign.httpclient.enabled`属性的值, 而且这里也没有设置havingValue, 根据源码可以知道， 如果没有设置havingValue, 那么这个属性的值会被和false进行比较

```java
//org.springframework.boot.autoconfigure.condition.OnPropertyCondition.Spec#isMatch
// 这里的requiredValue是havingValue
private boolean isMatch(String value, String requiredValue) {
	if (StringUtils.hasLength(requiredValue)) {
		return requiredValue.equalsIgnoreCase(value);
	}
	return !"false".equalsIgnoreCase(value);
}

```

搞半天这个Configurtion相当于没起作用。 


好好好，这么玩是吧。

既然这个配置不生效，那肯定有其他配置生效，我就找找其他配置，最终我在spring-cloud-openfeign-core这个jar包的loadbalancer这个包下面找到了我想要的配置

```java
@ConditionalOnClass(Feign.class)
@ConditionalOnBean({ LoadBalancerClient.class, LoadBalancerClientFactory.class })
@AutoConfigureBefore(FeignAutoConfiguration.class)
@AutoConfigureAfter({ BlockingLoadBalancerClientAutoConfiguration.class, LoadBalancerAutoConfiguration.class })
@EnableConfigurationProperties(FeignHttpClientProperties.class)
@Configuration(proxyBeanMethods = false)
// Order is important here, last should be the default, first should be optional
// see
// https://github.com/spring-cloud/spring-cloud-netflix/issues/2086#issuecomment-316281653
@Import({ HttpClientFeignLoadBalancerConfiguration.class, OkHttpFeignLoadBalancerConfiguration.class,
		HttpClient5FeignLoadBalancerConfiguration.class, DefaultFeignLoadBalancerConfiguration.class })
public class FeignLoadBalancerAutoConfiguration {

}

```
因为我们项目是采用springcloud alibaba进行开发，所以引入了spring-cloud-loadbalancer这个包,因此这个这个配置类就会生效，由于我们没有配置使用httpclient,同样也未使用okhttp,所以生效的配置类只有一个，那就是 `DefaultFeignLoadBalancerConfiguration`

这个配置类中retryClient会被加载,因为我们引入了spring-retry.

```java
@Bean
@ConditionalOnMissingBean
@ConditionalOnClass(name = "org.springframework.retry.support.RetryTemplate")
@ConditionalOnBean(LoadBalancedRetryFactory.class)
@ConditionalOnProperty(value = "spring.cloud.loadbalancer.retry.enabled", havingValue = "true",
		matchIfMissing = true)
public Client feignRetryClient(LoadBalancerClient loadBalancerClient,
		LoadBalancedRetryFactory loadBalancedRetryFactory, LoadBalancerClientFactory loadBalancerClientFactory) {
	return new RetryableFeignBlockingLoadBalancerClient(new Client.Default(null, null), loadBalancerClient,
			loadBalancedRetryFactory, loadBalancerClientFactory);
}
```

这也是为什么上面我们自己配置了自己的Client后，访问其他spring cloud服务会找不到地址，这是因为默认的client不会去通过LoadBalancer去获取服务地址。 

### 小插曲

期间debug的时候,还发现最终的Client的SeataFeignClient,我一看才发现某个公共包引入了Seata,但是没有使用Seata功能,然后Seata会把我们最终使用的FeignClient在给封装一次，所以后面我就把seata从项目中移除了。

### 解决方案

既然问题找到了,那么就好修改了，修改方式有两种，一种是创建自己的`RetryableFeignBlockingLoadBalancerClient`, 就把上面的代码拿过来抄一遍，只是自己指定SSLContext，另一种是启用httpclient

#### 方案一

```java
@Bean
public Client feignRetryClient(LoadBalancerClient loadBalancerClient,
                               LoadBalancedRetryFactory loadBalancedRetryFactory, LoadBalancerClientFactory loadBalancerClientFactory) throws NoSuchAlgorithmException, KeyManagementException {
    SSLContext ctx = SSLContext.getInstance("SSL");
    X509TrustManager tm = new X509TrustManager() {
        @Override
        public void checkClientTrusted(X509Certificate[] chain, String authType) {
        }
        @Override
        public void checkServerTrusted(X509Certificate[] chain, String authType) {
        }
        @Override
        public X509Certificate[] getAcceptedIssuers() {
            return null;
        }
    };
    ctx.init(null, new TrustManager[]{tm}, null);


    return new RetryableFeignBlockingLoadBalancerClient(new Client.Default(ctx.getSocketFactory(), (hostname, session) -> true), loadBalancerClient,
            loadBalancedRetryFactory, loadBalancerClientFactory);
}
```

#### 方案二
另一种方案就是启用httpclient,并且禁用ssl验证，配置如下

```yml
feign:
  httpclient:
    enabled: true
    disable-ssl-validation: true

```

自此这个问题解决了，当然在使用中更加倾向使用方案二，因为Feign默认的Client采用的是`HttpURLConnection`,它没有连接池,当然你也可以使用okhttp。

### 写到最后

这个问题看起来简单，但是排查起来还是颇费心思,很多细节隐藏到了框架之下,所以我想看源码还是有好处的，因为网上的文章别人的情况可能和你不一样，与其遨游在各个文章里面，还不如debug一把。

