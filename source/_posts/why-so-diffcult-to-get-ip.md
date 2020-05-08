---
title: 为什么获取请求IP这么麻烦？
date: 2020-05-06 18:12:55
tags: [HTTP]
---

web开发中，经常会获取请求端IP地址,熟悉的同学可能第一时间就想到了

```java
String ip = httpServletRequest.getRemoteAddr();
```

<!--more-->

如果你的客户端和你的服务器是直连的，中间没有经过任何的代理这样是没有问题，如果你是通过了代理服务器访问了后端服务，那么获取到的ip其实是代理服务器的ip。

```
直连: client --> Server
代理: client --> Nginx --> Server
```

所以经常会在项目中看到如下获取ip地址的代码:

```java
public static String getIpAddress(HttpServletRequest request) {
    String xIp = request.getHeader("X-Real-IP");
    String xFor = request.getHeader("X-Forwarded-For");
    if (StringUtils.isNotEmpty(xFor) && !"unKnown".equalsIgnoreCase(xFor)) {
      // 多次反向代理后会有多个ip值，第一个ip才是真实ip
      int index = xFor.indexOf(",");
      if (index != -1) {
          return xFor.substring(0, index);
      } else {
          return xFor;
      }
    }
    xFor = xIp;
    if (StringUtils.isNotEmpty(xFor) && !"unKnown".equalsIgnoreCase(xFor)) {
        return xFor;
    }
    if (StringUtils.isBlank(xFor) || "unknown".equalsIgnoreCase(xFor)) {
        xFor = request.getHeader("Proxy-Client-IP");
    }
    if (StringUtils.isBlank(xFor) || "unknown".equalsIgnoreCase(xFor)) {
        xFor = request.getHeader("WL-Proxy-Client-IP");
    }
    if (StringUtils.isBlank(xFor) || "unknown".equalsIgnoreCase(xFor)) {
        xFor = request.getHeader("HTTP_CLIENT_IP");
    }
    if (StringUtils.isBlank(xFor) || "unknown".equalsIgnoreCase(xFor)) {
        xFor = request.getHeader("HTTP_X_FORWARDED_FOR");
    }
    if (StringUtils.isBlank(xFor) || "unknown".equalsIgnoreCase(xFor)) {
        xFor = request.getRemoteAddr();
    }
    return xFor;
}
```

你会发现仅仅只是获取一个ip而已，竟然如此复杂。莫急，且听我细细道来。

接下来我们分析下为什么经过了代理之后获取真实ip变得如此复杂了。

### X-Forwarded-For

X-Forwarded-For(XFF) 是一个 HTTP 扩展头部。HTTP/1.1(RFC 2616)协议并没有对它的定义，它最开始是由 Squid 这个缓存代理软件(关于squid的使用，可以参考我这篇文章)引入，用来表示 HTTP 请求端真实 IP。如今它已经成为事实上的标准，被各大 HTTP 代理、负载均衡等转发服务广泛使用，并被写入 [RFC 7239](https://tools.ietf.org/html/rfc7239) (Forwarded HTTP Extension)标准之中。

比如我们一般在nginx中会这样配置

```nginx

http {
    include       mime.types;
    default_type  application/octet-stream;

    # 指定上游服务器(可以指定多个，还可以指定权重等)，并取名为local
    upstream local {
        server 127.0.0.1:8001;
    }
    server {
      listen       8002;
      server_name  localhost;

      location / {
        # 由于有了反向代理，A-->B-->C这样的请求链，所以为了让C拿到A的信息就进行如下的配置，这些特性可以在http-proxy-module中看到
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;

        # 将请求代理到上游服务器
        proxy_pass http://local;
      }
}
```

X-Forwarded-For 请求头格式非常简单，就这样：

```shell
X-Forwarded-For: client, proxy1, proxy2
```

XFF的格式非常简单,它的内容由 "英文逗号 + 空格" 隔开的多个部分组成，最开始的是离服务端最远的设备 IP(client ip)，然后是每一级代理设备的 IP。
> 所以上面代码中从XFF中获取IP时,会获取逗号分隔后的第一个值


比如我们有这样的请求链条: client --> proxy1 --> proxy2 --> proxy3 --> server。请求经过了proxy1, proxy2, proxy3三个代理,其ip分别是proxy1IP, proxy2IP, proxy3IP,而客户端真实ip为clientIP,那么根据XFF标准,server最终会收到以下信息

```shell
X-Forwarded-For: clientIP, proxy1IP, proxy2IP
```

proxy3直连server,它会给XFF追加proxy2IP,表示它是在帮proxy2转发请求，而proxy3IP没有出现在XFF中,它可以在服务端通过 Remote Address 字段获得。我们知道 HTTP 连接基于 TCP 连接，HTTP 协议中没有 IP 的概念，Remote Address 来自 TCP 连接，表示与服务端建立 TCP 连接的设备 IP，在这个例子里就是 proxy3IP。

Remote Address 无法伪造，因为建立 TCP 连接需要三次握手，如果伪造了源 IP，无法建立 TCP 连接，更不会有后面的 HTTP 请求。不同语言获取 Remote Address 的方式不一样，例如 php 是 `$_SERVER["REMOTE_ADDR"]`，java 中 是 `request.getRemoteAddr()`，但原理都一样。



我们还知道上面的代码中总是出现了一个词 unknown ? 为什么总是使用它来判断呢？

这是因为squid.conf的配置文件中如果将 forwarded设置为 `forwarded_for off`,那么XFF的值就是unknown

### Proxy-Client-IP和WL-Proxy-Client-IP

Proxy-Client-IP 字段和 WL-Proxy-Client-IP 字段只在 Apache（Weblogic Plug-In Enable）+ WebLogic 搭配下出现，这里的WL就是WebLogic的缩写。


### X-Real-IP, HTTP_CLIENT_IP, HTTP_X_FORWARDED_FOR

这些都是代理服务器自定义的header,比如X-Real-IP一般nginx会加上, HTTP_X_FORWARDED_FOR 可以认为就是 X_FORWARDED_FOR。 比如我nginx配置如下：

```nginx

http {

  include       mime.types;
  default_type  application/octet-stream;
  underscores_in_headers on;

  server {
    listen       127.0.0.1:8001;
    server_name  localhost;
   
    location / {
        return 200 'clintIp=$http_client_ip,xff=$HTTP_X_FORWARDED_FOR';
    }
  }
}


```

当你这样访问nginx服务时:

```shell
$ curl  127.0.0.1:8001/ -H 'x-forwarded-for: 1.2.2.2' -H 'client-ip: 1.1.1.1'

  clintIp=1.1.1.1,xff=1.2.2.2

```

上面用到的`$http_client_ip`,`$HTTP_X_FORWARDED_FOR`(不区分大小写),这种是nginx中获取用户自定义的header的写法(header名称均为小写,且以http开头,以下划线连接)。而我们在访问服务器的时候，是可以传递任何header的。这也就意味着你获取到的header值不是完全可信的。

所以，如果我们的Web应用是直接对外提供服务的，那么在进行安全相关的操作时，只能通过Remote Address获取IP,不能相信任何请求头

如果我们的Web应用是通过Nginx等Web Server进行反向代理后对外提供服务的，想要获取client ip需要使用X-Forwarded-For第一节或X-Real-IP来获取IP(此时Remote Address 获取的是Nginx 所在服务器的IP,当然第一节很容易被伪造,如果拿后面部分的地址,拿到的实际是Nginx所在服务器的IP,不过这样一定可以知道是从Nginx转发过来的),同时还应该禁止Web服务应用直接对外提供服务。
