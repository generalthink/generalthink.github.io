---
title: api-design.md
date: 2025-01-03 10:15:56
tags: [SpringMVC, 架构]
---

最近改造了一个老项目，遇到了两个问题，一个是接口响应混乱,另一个是版本控制混乱，给开发和重构带来了巨大的不变，于是整理了下问题以及是如何解决的。


### 接口混乱

比如用户approve某个证书请求后，我们需要更新请求状态，然后将证书上传到第三方认证系统中去。 
这个时候接口返回数据如下

```json
{
"success": true,
"code": 10001,
"info": "TE service not working",
"message": "OK",
"data": {
	"status": "Failed",
	"TeId": -1
	}
}
```
看success表示请求成功，看这个info又明显提示TE Service有问题， 然后这个data呢，看起来返回的数据也是错的。 

后面我清理了下这个逻辑,发现实际上是状态变更成功了，但是将证书发送到TE失败了，因为TE服务这个时候不可用，所以code又返回了10001。

而且如果当状态变更时数据校验不过的时候返回的json数据又变成了这样子

```json
{
"success": false,
"code": 0,
"info": "",
"message": "current user is illegal",
"data": {
	"status": "Success",
	"TeId": 0
	}
}

```

对于这个情况为什么code返回的是0, message又提示有问题，data返回的数据看上去又是正常， 这里的info和message有啥区别呀？


最后看了代码才知道code, info, data是调用第三方服务的返回结果， success和message又是这个状态变更的处理结果。 


可以看到这个返回的逻辑太过于混乱了， 不统一，如果不统一下response,后面做功能可是在太痛苦了。

于是经过讨论， 我把这个返回值定义成如下结构

```java
@Data
public class ApiResponse<T> {
	private boolean success;
	private T data;
	private int code;
	private String message;
}

```
然后解析流程如下

！[解析流程](/images/api-response-process.png)


### 版本控制混乱

接口多版本定义不明确，我梳理了下我看到的集中定义方式

1. /v1/api/user/list
2. /api/user/list/v2
3. /api/company/list?version=2
4. 通过http header  X-API-VERSION来定义


这几种方式URL path最直观也不容易出错,就是写法各种各样。 放到请求参数中又不容易携带，有侵入性，HTTP头方式没有侵入性，如果是部分接口需要那可以考虑下这种。

我的思路是在使用层面来进行统一，不让它侵入url中,开发的时候要更加易于使用，刚我之前的文章提到了RequestMappingHandlerMapping就是处理@RequestMapping注解的，所以我们可以自定义handlerMapping来处理

```java

@Target({ElementType.METHOD, ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
public @interface ApiVersion {
String[] value();
}

public class ApiVersionRequestMapping extends RequestMappingHandlerMapping {

	// springmvc 5.3开始有这个类
    private final PathPatternParser pathPatternParser = new PathPatternParser();
    @Override
    protected void registerHandlerMethod(Object handler, Method method, RequestMappingInfo mapping) {
        Class<?> controllerClass = method.getDeclaringClass();
        ApiVersion apiVersion = AnnotationUtils.findAnnotation(controllerClass, ApiVersion.class);
        ApiVersion methodAnnotation = AnnotationUtils.findAnnotation(method, ApiVersion.class);
        // method上的version会覆盖class上面的
        if (methodAnnotation != null) {
            apiVersion = methodAnnotation;
        }
        String[] versions = apiVersion != null ? apiVersion.value() : new String[0];

        if (versions.length > 0) {
            for (String version : versions) {
                String versionedPath = "/" + version;
                RequestMappingInfo newMapping = RequestMappingInfo
                        .paths(versionedPath)
                        .options(this.getBuilderConfiguration())
                        .build().combine(mapping);
                super.registerHandlerMethod(handler, method, newMapping);
            }
        } else {
            // 如果没有 ApiVersion 注解，使用原始的 mapping
            super.registerHandlerMethod(handler, method, mapping);
        }
    }

    @Override
    public RequestMappingInfo.BuilderConfiguration getBuilderConfiguration() {
        RequestMappingInfo.BuilderConfiguration config = super.getBuilderConfiguration();
        config.setPatternParser(pathPatternParser);
        return config;
    }
}

```

> 会在初始化的时候就扫描handler,入口可以看这里： org.springframework.web.servlet.handler.AbstractHandlerMethodMapping#afterPropertiesSet

需要注意的是需要把我们自定的mapping加入到SpringMVC中

```java
@Configuration
public class AppConfig implements WebMvcRegistrations {

    @Override
    public RequestMappingHandlerMapping getRequestMappingHandlerMapping() {
        return new ApiVersionRequestMapping();
    }
}

```
现在我们就实现了下面的功能了。

```java
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
@RequestMapping("/test")
public class TestController {

    @ApiVersion("v1")
    @RequestMapping("/user")
    public String getUserV1() {
        return "This is API version 1";
    }

    @ApiVersion("v2")
    @RequestMapping("/user")
    public String getUserV2() {
        return "This is API version 2";
    }

    @RequestMapping("/user")
    public String getUserDefault() {
        return "This is the default API version";
    }
}

```

1. /v1/test/user 将访问 getUserV1() 方法
2. /v2/test/user 将访问 getUserV2() 方法
3. /test/user 将访问 getUserDefault() 方法