---
title: springcloud-integrate-nacos-as-discovering
date: 2025-03-03 17:08:43
tags: [SpringCloud, Nacos, SpringBoot]
---

这边文章主要探讨Nacos提供的服务发现功能如何和SpringCloud进行集成的。


### 集成服务发现

首先需要引入我们的starter
```xml
 <dependency>
    <groupId>com.alibaba.cloud</groupId>
    <artifactId>spring-cloud-starter-alibaba-nacos-discovery</artifactId>
    <version>2021.1</version>
</dependency>
```
 添加配置

 ```yml
spring:
  cloud:
    nacos:
      discovery:
        #默认为true,不配置也行
        enable: true
        # 服务注册地址
        server-addr: localhost:8848

 ```
<!--more-->

### 源码入口

Nacos中集成的入口同样是通过`spring-cloud-starter-alibaba-nacos-discovery-2021.1.jar!\META-INF\spring.factories`其中指定了自动加载的配置类

```properties
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
  com.alibaba.cloud.nacos.discovery.NacosDiscoveryAutoConfiguration,\
  com.alibaba.cloud.nacos.endpoint.NacosDiscoveryEndpointAutoConfiguration,\
  com.alibaba.cloud.nacos.registry.NacosServiceRegistryAutoConfiguration,\
  com.alibaba.cloud.nacos.discovery.NacosDiscoveryClientConfiguration,\
  com.alibaba.cloud.nacos.discovery.reactive.NacosReactiveDiscoveryClientConfiguration,\
  com.alibaba.cloud.nacos.discovery.configclient.NacosConfigServerAutoConfiguration,\
  com.alibaba.cloud.nacos.NacosServiceAutoConfiguration
```

### 具体分析

`NacosServiceAutoConfiguration`中初始化了`nacosServiceManager`,从名字可以看出它的主要作用

```java
@Bean
  public NacosServiceManager nacosServiceManager() {
    return new NacosServiceManager();
  }
```

而在NacosServiceManager中维护了一个很重要的变量NamingService; 这个接口定义了服务注册，服务获取，服务的下线等方法,当然具体的调用它也是委托给了`NamingClientProxyDelegate`的


`NacosDiscoveryAutoConfiguration`中初始化了`NacosServiceDiscovery`,而它就被用于与Nacos服务发现系统进行交互,当然获取已注册的服务列表，获取某个服务可用的实例列表等方法其实都是委托给`NacosServiceManager`实现的。

```java
@Bean
  @ConditionalOnMissingBean
  public NacosServiceDiscovery nacosServiceDiscovery(
      NacosDiscoveryProperties discoveryProperties,
      NacosServiceManager nacosServiceManager) {
    return new NacosServiceDiscovery(discoveryProperties, nacosServiceManager);
  }
    
```



`NacosDiscoveryClientConfiguration`就负责初始化DiscoveryClient,这是spring cloud的用于服务发现的基本接口，所以服务发现的组件都要实现这个接口，比如Netflix,consul

```java
@Bean
  public DiscoveryClient nacosDiscoveryClient(
      NacosServiceDiscovery nacosServiceDiscovery) {
    return new NacosDiscoveryClient(nacosServiceDiscovery);
  }
```

到这里我们基本看出了其主要类的封装线路是怎样的

```
NacosDiscoveryClient --> NacosServiceDiscovery --> NacosServiceManager --> NacosNamingService
--> NamingClientProxyDelegate-->NamingHttpClientProxy,NamingGrpcClientProxy(符合和Nacos Server进行通信)
```

而基本我们和nacos server的请求都是通过NacosNamingService来实现的。


### 服务的注册以注销


`NacosServiceRegistryAutoConfiguration`中初始化了`NacosServiceRegistry`,而它就是用于服务的register,deregister了。

```java
  @Bean
  public NacosServiceRegistry nacosServiceRegistry(
      NacosDiscoveryProperties nacosDiscoveryProperties) {
    return new NacosServiceRegistry(nacosDiscoveryProperties);
  }
  @Bean
  @ConditionalOnBean(AutoServiceRegistrationProperties.class)
  public NacosAutoServiceRegistration nacosAutoServiceRegistration(
      NacosServiceRegistry registry,
      AutoServiceRegistrationProperties autoServiceRegistrationProperties,
      NacosRegistration registration) {
    return new NacosAutoServiceRegistration(registry,
        autoServiceRegistrationProperties, registration);
  }

```

`NacosAutoServiceRegistration`会在web server初始化完成的时候调用判断当前服务是否需要注册,因为有的服务只需要访问其他服务，而不需要其他服务访问它,这个参数可以这样配置

```
# 默认为true
spring.cloud.nacos.discovery.register-enabled=false
```

```java
@Override
  protected void register() {
    if (!this.registration.getNacosDiscoveryProperties().isRegisterEnabled()) {
      log.debug("Registration disabled.");
      return;
    }
    if (this.registration.getPort() < 0) {
      this.registration.setPort(getPort().get());
    }
    // 最终调用NacosServiceRegistry#register方法
    super.register();
  }

```


而NacosServiceRegistry中实际上也注入了NacosServiceManager,它会在web server初始化完成的时候被调用，从而保证服务当前服务注册到Nacos中

```java
  @Override
  public void register(Registration registration) {
    // 省略不必要代码
    NamingService namingService = namingService();
    String serviceId = registration.getServiceId();
    String group = nacosDiscoveryProperties.getGroup();

    // 封装当前服务ip,port等配置封装成Instance
    Instance instance = getNacosInstanceFromRegistration(registration);

    // 服务注册
    namingService.registerInstance(serviceId, group, instance);
    log.info("nacos registry, {} {} {}:{} register finished", group, serviceId,
          instance.getIp(), instance.getPort());
  }

```

同样的，当容器销毁,那么服务就会被注销,deregister方法会被调用
```java
  @Override
  public void deregister(Registration registration) {

    // 忽略部分无关代码

    log.info("De-registering from Nacos Server now...");


    NamingService namingService = namingService();
    String serviceId = registration.getServiceId();
    String group = nacosDiscoveryProperties.getGroup();

   
    namingService.deregisterInstance(serviceId, group, registration.getHost(),
          registration.getPort(), nacosDiscoveryProperties.getClusterName());
   
    log.info("De-registration finished.");
  }

```


如果你往namingService的方法输入去看的时候，你会发现它会根据ephemeral的变量的值(默认为true)来决定是使用GRPC还是HTTP请求

```
# false为永久实例，true表示临时实例
spring.cloud.nacos.discovery.ephemeral=false
```
上面是基于Spring Cloud进行配置，false为永久实例，true表示临时实例，默认为true。

目前，无论是Nacos 1.x版本，还是2.x版本，ephemeral的默认值都是true。在1.x版本中服务注册默认采用http协议，2.x版本默认采用grpc协议，但这都未影响到ephemeral字段的默认值。
也就是说，一直以来，Nacos实例默认都是以临时实例的形式进行注册的。

>临时实例向Nacos注册，Nacos不会对其进行持久化存储，只能通过心跳方式保活。默认模式是：客户端心跳上报Nacos实例健康状态，默认间隔5秒，Nacos在15秒内未收到该实例的心跳，则会设置为不健康状态，超过30秒则将实例删除。持久化实例向Nacos注册，Nacos会对其进行持久化处理。当该实例不存在时，Nacos只会将其健康状态设置为不健康，但并不会对将其从服务端删除。另外，可以使用实例的ephemeral来判断健康检查模式，ephemeral为true对应的是client模式（客户端心跳），为false对应的是server模式（服务端检查）


### 心跳

心跳从2.0开始就采用GRPC的方式来进行心跳检测，也就是由服务器端和客户端通过长链接的方式来确保心跳,当我们初始化

`NamingClientProxyDelegate`的时候会初始化`NamingGrpcClientProxy`,其中会调用RpcClient#start方法


```java
// 线程次任务一直运行
clientEventExecutor.submit(() -> {
            while (true) {
                try {
                    if (isShutdown()) {
                        break;
                    }
                    ReconnectContext reconnectContext = reconnectionSignal
                            .poll(keepAliveTime, TimeUnit.MILLISECONDS);
                    if (reconnectContext == null) {
                        // check alive time.
                        // 判断举例上一次检查的时间是否大于等于5s(keepAliveTime)
                        if (System.currentTimeMillis() - lastActiveTimeStamp >= keepAliveTime) {
                            boolean isHealthy = healthCheck();
                            if (!isHealthy) {
                                if (currentConnection == null) {
                                    continue;
                                }
                                LoggerUtils.printIfInfoEnabled(LOGGER,
                                        "[{}] Server healthy check fail, currentConnection = {}", name,
                                        currentConnection.getConnectionId());
                                
                                 RpcClientStatus rpcClientStatus = RpcClient.this.rpcClientStatus.get();
                                if (RpcClientStatus.SHUTDOWN.equals(rpcClientStatus)) {
                                    break;
                                }
                                
                                boolean statusFLowSuccess = RpcClient.this.rpcClientStatus
                                        .compareAndSet(rpcClientStatus, RpcClientStatus.UNHEALTHY);
                                if (statusFLowSuccess) {
                                    //不健康，等待重连,设置onRequestFail=false
                                    reconnectContext = new ReconnectContext(null, false);
                                } else {
                                    continue;
                                }
                                
                            } else {
                                // 更新上一次active时间戳
                                lastActiveTimeStamp = System.currentTimeMillis();
                                continue;
                            }
                        } else {
                            continue;
                        }
                        
                    }
                    
                    //省略其他代码
                    // 如果onRequestFail则重新连接
                    reconnect(reconnectContext.serverInfo, reconnectContext.onRequestFail);
                } catch (Throwable throwable) {
                    // Do nothing
                }
            }
        });

```
而healthCheck()方法就是发送一个空请求

```java
 private boolean healthCheck() {
        HealthCheckRequest healthCheckRequest = new HealthCheckRequest();
        if (this.currentConnection == null) {
            return false;
        }
        try {
            Response response = this.currentConnection.request(healthCheckRequest, 3000L);
            // not only check server is ok, also check connection is register.
            return response != null && response.isSuccess();
        } catch (NacosException e) {
            // ignore
        }
        return false;
    }

```

而Nacos服务器接收到这个请求后，就会后台任务来剔除不健康的连接,有兴趣的可以去看看`ConnectionManager`这个类。


### 8848, 9848, 9849端口

Nacos 2.x默认使用的端口为8848（HTTP管理端口）、9848（客户端gRPC请求服务端端口, 8848+1000）和9849（服务端gRPC请求服务端端口, 8848+1001）。客户端在连接时，虽然主要配置的是管理端访问端口8848，但实际上客户端会根据服务端的配置自动计算其他端口进行通信。

我们可以从nacos的启动日志中看到

```
2025-03-26 10:18:28,570 INFO Nacos GrpcSdkServer Rpc server starting at port 9848

2025-03-26 10:18:30,646 INFO Nacos GrpcSdkServer Rpc server started at port 9848

2025-03-26 10:18:30,651 INFO Nacos GrpcClusterServer Rpc server starting at port 9849

2025-03-26 10:18:30,656 INFO Nacos GrpcClusterServer Rpc server started at port 9849

```

### 总结

源码的分析过程中我们会发现底层是Nacos原生代码，上面就是会写各种疯转来结合SpringCloud,然后对于存在多种实现的还会使用代理设计模式，这也值得我们借鉴。