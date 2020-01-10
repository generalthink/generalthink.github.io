---
title: 对于Http Status Code,我有话说
date: 2020-01-08 17:20:36
tags: HTTP
---

现在很多项目都是web项目，前后端分离，唯一的交互就是通过restful接口,而当我们请求返回的时候,status code如何返回呢?

首先介绍下常用的http status code有哪些。


### 2XX(Success 成功状态码)

#### 200 - OK

请求成功

#### 201 - Created

文档创建成功,**比如新增一个user成功**

#### 202 - Accepted

请求已被接受，但相应的操作可能尚未完成。这用于后台操作，**例如数据库压缩等异步操作**


### 4XX(Client Error 客户端错误状态码)

#### 400 - Bad Request

请求参数有误(**比如应该传一个Number类型的参数，你却传了一个字符串**),请求无法被服务器理解,修改后可以重新提交这个请求

<!--more-->

#### 401 - Unauthorized

当前请求用户未被授权,**比如未登陆**

#### 403 - Forbidden

当前请求被拒绝。**比如文件系统访问权限有问题，或者进行了越权操作(比如普通用户试图获取admin用户列表)**

#### 404 - Not Found

无法找到请求资源,**一般是url错误**

#### 405 - Method Not Allowed

使用无效的HTTP请求类型对请求的URL进行了请求。**比如某个api只支持post,而client却使用了get**

#### 406 - Not Acceptable

服务器不支持请求的content type

#### 413 - Request Entity Too Large

请求体太大不支持，**一般是上传的文件超出了限定导致的。**

### 5XX(Server Error 服务器错误状态码)

#### 500 - Internal Server Error

表示服务端在执行请求时发生了错误。 **可能是服务器或者应用存在bug**

#### 503 - Service Unavailable

服务不可用，现在无法处理请求。


### 返回什么样的错误码

一般在restful API里，我们对于状态码的认定是这样的：

**1. 2xx: server 收到 client 端请求，可以执行**
**2. 4xx: client 送來资料有错，server 端无法执行 (client 修正错误后，可再送一次请求)**
**3. 5xx: client 送來的资料没错，但 server 端出错无法执行(client 端无法 take 任何 action)**

但是**我也见过status code返回200,然后在response中通过自定义code告诉用户请求是否成功**

```json
{
  "code": 10020,
  "msg": "name不能为空"
}
```

**虽然萝卜青菜各有所爱,我个人仍然认为对于REST接口，就应该用HTTP层面的status来表达操作是否成功。**,因为网络通讯不仅仅是一个客户端一个服务器的事情,中间会有很多层节点，如果其中某一层对status 200的请求理解非常正统,做了cache或者什么处理,那你可能就坐蜡了。而且既然是写接口，就最好标准一点,别给自己埋坑。

所以对于成功的请求就应该返回2XX,对于4XX接口,你可以做对应的封装

![返回数据](/images/java/response.png)

状态码一定要返回4XX,但是你可以在reponse data中返回异常错误码(这个错误码是业务错误的错误码),这个时候可以和前端约定好这个业务错误码代表什么意思,就可以给用户相应的提示了。


### 返回的数据格式

```java
public class BaseResponse<T> {

    // 业务异常码
    private int code;

    // 说明信息
    private String msg;

    // 需要封装返回的数据
    private T data;
}
```
比如对于`GET /api/users/1`,这样的请求返回的数据就放在data这个字段当中。

```java
{
  "code":200,
  "msg":"SUCCESS",
  "data": {
    "id":1095,
    "name":"张三"
  }
}

```

而对于请求失败的错误(http status code是4XX),则是会返回下面的数据
```java
{
  "code": 10030,
  "msg": "id不存在"
}
```

### Spring对Http Status Code的支持

```java

@ResponseStatus(HttpStatus.CREATED)
public BaseResponse addUser(UserParam userParam) {
  
    return service.addUser(userParam);
}

```
可以直接通过`ResponseStatus`注解来指定返回的错误码是多少。当然上面说的是正常情况,那些参数失败校验失败或者是其他业务异常导致的情况如何处理呢？

有两种办法，第一种是通过Spring的 ResponseEntity 这个类来完成

```java
public ResponseEntity<BaseResponse> addUser(UserParam userParam) {
  
  if(validate(userParam)) {
      return service.addUser(userParam);
  }

  return  new ResponseEntity<>(ResponseUtil.fail(STATUS_INVALID_PARAM), HttpStatus.NOT_FOUND);
}

```

第二种方式是借助全局异常处理,如果在service中发现客户端传递过来的参数存在问题，可以通过抛出异常,然后在全局异常拦截器中进行统一处理

```java
@RestControllerAdvice
public class WebExceptionConfig {

  @ExceptionHandler(InvalidParamException.class)
  @ResponseStatus(HttpStatus.BAD_REQUEST)
  public BaseResponse handleInvalidParamException(InvalidParamException e) {
      return ResponseUtil.fail(STATUS_INVALID_PARAM, e.getMessage());
  }
}

```

### 总结

1. 不同的情况返回不同的status code,对于失败的请求,请返回4XX,并可以通过返回数据指定业务异常code
2. 内部一定要统一
3. 不要让返回的status code影响你的业务逻辑


### 点关注，不迷路

如果觉得我写的还行的话，请给我个点赞、关注、分享呀，对我是很大的激励呀。

> 公众号 think123,可以第一时间阅读哟。