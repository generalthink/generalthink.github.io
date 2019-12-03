---
title: k8s rolling update遇到的一个问题
date: 2019-12-02 18:02:24
tags:
- K8S
- Rolling Update
---



**在使用K8S rolling update的时候,我同时使用JMeter不间断call API，就有少部分请求抛出了这个错误**

```
{
  "timestamp":"2019-12-02T07:33:47.120+0000",
  "status":500,
  "error":"Internal Server Error",
  "message":"No message available",
  "path":"/api/anon/article/TW1Tq/feed"
}
```

<!--more-->

上面的异常是SpringBoot的BasicErrorController抛出的带有错误，HTTP状态和异常消息的详细信息的JSON响应

```java
package org.springframework.boot.autoconfigure.web.servlet.error;

//...

@Controller
@RequestMapping("${server.error.path:${error.path:/error}}")
public class BasicErrorController extends AbstractErrorController {

  //...
  @RequestMapping
  public ResponseEntity<Map<String, Object>> error(HttpServletRequest request) {

    // 方法内部组装了timestamp,error,message,path等信息
    Map<String, Object> body = getErrorAttributes(request,
        isIncludeStackTrace(request, MediaType.ALL));
    HttpStatus status = getStatus(request);
    return new ResponseEntity<>(body, status);
  }

}

```
这个错误显然是因为服务更新中，容器服务被直接停止，部分请求仍被分发到终止的容器，导致服务出现500错误，这部分错误请求数据占用比较少。由于是部分请求会产生服务器错误的情况，考虑使用优雅的终止方式，将错误请求降到最低，直至滚动更新不影响用户。

>kubernetes终止pod过程如下
1. kubernetes停止将新的连接路由到需要终止的pod。已建立的连接将保持不变并保持打开状态。
2. 假设容器将开始停止，Kubernetes将TERM信号发送到Pod中每个容器的根进程。它发送的这个信号无法配置。
3. 等待pod的TerminalGracePeriodSeconds中指定的时间段(默认为30秒),如果此时容器仍在运行，它将发送KILL终止容器


由于在更新过程中,我们使用jmeter一直发起请求，所以有部分请求仍会发送到需要delete的pod,很明显在30s内它处理不完这么多的请求,而当过了30s之后，发送KILL命令就导致有些请求失败了。

那么能不能当我们收到通知的时候就不接收新的请求了呢？然后再把正在处理的请求处理完成之后在关闭我们的应用。

由于我们使用的内嵌的tomcat,所以只需要实现对应的接口就行了。

```java

import org.apache.catalina.connector.Connector;
import org.apache.tomcat.util.threads.ThreadPoolExecutor;
import org.springframework.boot.web.embedded.tomcat.TomcatConnectorCustomizer;
import org.springframework.context.ApplicationListener;
import org.springframework.context.event.ContextClosedEvent;

@Slf4j
public class GracefulShutdown implements TomcatConnectorCustomizer,
        ApplicationListener<ContextClosedEvent> {

  private static final int TIMEOUT = 30;

  private volatile Connector connector;

  @Override
  public void customize(Connector connector) {
      this.connector = connector;
  }

  @Override
  public void onApplicationEvent(ContextClosedEvent event) {
    this.connector.pause();
    Executor executor = this.connector.getProtocolHandler().getExecutor();
    if (executor instanceof ThreadPoolExecutor) {
      try {

        ThreadPoolExecutor threadPoolExecutor = (ThreadPoolExecutor) executor;

        // 将状态设置为shutdown,不再接收新的请求，正在跑的任务会执行完
        threadPoolExecutor.shutdown();

        if (!threadPoolExecutor.awaitTermination(TIMEOUT, TimeUnit.SECONDS)) {

          log.warn("Tomcat thread pool did not shut down gracefully within "
                  + TIMEOUT + " seconds. Proceeding with forceful shutdown");

          // 如果在规定的时间30s之内还未处理完成，那么直接关闭
          threadPoolExecutor.shutdownNow();

          if (!threadPoolExecutor.awaitTermination(TIMEOUT, TimeUnit.SECONDS)) {
              log.error("Tomcat thread pool did not terminate");

          }
        }
      } catch (InterruptedException ex) {
          Thread.currentThread().interrupt();
      }
    }
  }

}

@Configuration
public class ShutdownConfiguration {

  @Bean
  public GracefulShutdown gracefulShutdown() {
      return new GracefulShutdown();
  }

  @Bean
  public ConfigurableServletWebServerFactory webServerFactory(final GracefulShutdown gracefulShutdown) {
      TomcatServletWebServerFactory factory = new TomcatServletWebServerFactory();
      factory.addConnectorCustomizers(gracefulShutdown);
      return factory;
  }
}

```

通过上面的设置我们就能实现所说的效果。同时即使在滚动更新的时候，一直有请求不断的call api，也不会出现报错的问题了。
>这里不断call api,类似于一个高并发请求(现在不说点高并发都不好意思说自己是写代码的了,狗头)



