---
title: Java中如何为HTTP请求设置代理?
date: 2020-02-20 13:36:32
tags:
- java
- squid proxy
- kubernetes
---

### 什么是代理服务器

代理服务器充当你和Internet之间的网关，就像一个中间人。它实际上是一个中间服务器，可以将用户与它们游览的网站区分开。

如果你使用了代理服务器，那么网络流量会通过代理服务器流向你请求的地址。然后该请求通过同一台代理服务器返回,然后代理服务器将从网站接收到的数据转发给你。

当然如果仅仅是这样，也没什么必要使用代理服务器，我们直接访问网站岂不更美？

现在代理服务器的功能远不只是转发Web请求，而这一切都是为了保证数据安全和网络性能。代理服务器充当防火墙和Web筛选器，提供共享的网络连接，并缓存数据以加快常见请求的速度。
而且还可以保护用户和内部网络以免收到外部Internet的不良影响。

<!--more-->

### Java如何使用代理服务器

java 有两种方式可以设置代理服务器

#### 如何设置

1. 通过命令行选项进行设置
```
java -Dhttp.proxyHost=webcache.example.com -Dhttp.proxyPort=8080 -Dhttp.nonProxyHosts="localhost|host.example.com" test.jar 
```

所有http连接都将通过webcache.example.com上的代理服务器在端口8080上监听(如果不指定端口默认是80),此外，连接到localhost或host.example.com时将不使用代理。


2. 通过System.setProperty(String，String)方法
```java
// 设置代理
System.setProperty("http.proxyHost", "webcache.example.com");
System.setProperty("http.proxyPort", "8080");


// 下一个连接将会使用代理
URL url = new URL("http://java.example.org/");
InputStream in = url.openStream();

// 清除代理
System.clearProperty("http.proxyHost");

// 从现在开始，http连接将直接完成而不再使用代理
```

#### 参数说明

1. http.proxyHost : 代理服务器主机名
2. http.proxyPort : 端口号,默认是80
3. https.proxyHost : https代理服务器主机名
4. https.proxyPort: 代理端口号,默认是443
5. http.nonProxyHosts : 指定绕过代理的主机列表，使用 | 分割的模式列表,可以以通配符 * 开头或者结尾,任何匹配这些模式之一的主机都将通过直接连接而不是通过代理访问。该设置对http,https通用

> 其他比如ftp,socket等设置可以参考官方文档: https://docs.oracle.com/javase/8/docs/technotes/guides/net/proxies.html

### 使用代理服务器

我刚才有讲过,一般情况只是只是转发请求，我们是用不到代理服务器的，但是刚好我遇到了不一般的情况。
我们的服务部署在Kubernetes中,但是有些请求需要走内网访问内部服务,有些请求需要走外网下载文件。而且不能同时给它设置内网ip和public ip(同时设置,访问其他服务时，其他服务会认为是外网在访问)。

因此解决方案是将我们的服务部署在拥有内网访问的worker中,而将代理服务器部署在具有public ip的worker中,当某些访问需要访问外网的时候,走代理服务器即可。
代理服务器我们选择了squid这个老牌产品。

#### squid

首先部署squid的service

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: squid
spec:
  replicas: 1
  selector:
    matchLabels:
      run: squid
  template:
    metadata:
      labels:
        run: squid
    spec:
      nodeSelector:
        node.vks/internet-access: "true"
      volumes:
        - name: proxy-config
          configMap:
            name: proxy-configmap
      containers:
        - name: squid
          image: sameersbn/squid:3.5.27-2
          imagePullPolicy: IfNotPresent
          # 将squid的配置文件挂载到/etc/squid中
          volumeMounts:
            - name: proxy-config
              mountPath: /etc/squid
              readOnly: true
          ports:
            - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: proxy-service
spec:
  ports:
    - port: 80
  selector:
    run: squid
```

而proxy.conf的配置实际是默认的，你可以使用默认的，也可以使用这个我优化了的

```
http_port 80
# Example rule allowing access from your local networks.
# Adapt to list your (internal) IP networks from where browsing
# should be allowed
acl localnet src 10.0.0.0/8 # RFC1918 possible internal network
acl localnet src 172.16.0.0/12  # RFC1918 possible internal network
acl localnet src 192.168.0.0/16 # RFC1918 possible internal network
acl SSL_ports port 443
acl Safe_ports port 80    # http
acl Safe_ports port 21    # ftp
acl Safe_ports port 443   # https
acl Safe_ports port 70    # gopher
acl Safe_ports port 210   # wais
acl Safe_ports port 1025-65535  # unregistered ports
acl Safe_ports port 280   # http-mgmt
acl Safe_ports port 488   # gss-http
acl Safe_ports port 591   # filemaker
acl Safe_ports port 777   # multiling http
acl CONNECT method CONNECT
http_access deny CONNECT !SSL_ports
http_access allow localhost manager
http_access allow localnet
http_access deny manager
http_access allow all
# disable caching
cache deny all
cache_mem 8 MB
cache_dir null /tmp
# disable unnecessary logs
cache_log / dev/null
# To make it anonymous
forwarded_for off
request_header_access Allow allow all
request_header_access Authorization allow all
request_header_access WWW-Authenticate allow all
request_header_access Proxy-Authorization allow all
request_header_access Proxy-Authenticate allow all
request_header_access Cache-Control allow all
request_header_access Content-Encoding allow all
request_header_access Content-Length allow all
request_header_access Content-Type allow all
request_header_access Date allow all
request_header_access Expires allow all
request_header_access Host allow all
request_header_access If-Modified-Since allow all
request_header_access Last-Modified allow all
request_header_access Location allow all
request_header_access Pragma allow all
request_header_access Accept allow all
request_header_access Accept-Charset allow all
request_header_access Accept-Encoding allow all
request_header_access Accept-Language allow all
request_header_access Content-Language allow all
request_header_access Mime-Version allow all
request_header_access Retry-After allow all
request_header_access Title allow all
request_header_access Connection allow all
request_header_access Proxy-Connection allow all
request_header_access User-Agent allow all
request_header_access Cookie allow all
request_header_access All deny all
```

需要注意的是`node.vks/internet-access: "true"`,squid部署在了具有公网访问权限的worker中。

> 如果想要验证proxy是否可用，可以通过curl的方式(自行设定proxy)

#### 设定Java运行参数

```java
apiVersion: apps/v1
kind: Deployment
metadata:
  name: helloworld
spec:
  replicas: 1
  selector:
    matchLabels:
      run: helloworld
  template:
    metadata:
      labels:
        run: helloworld
    spec:
      nodeSelector:
        node.vks/intranet: "true"
      containers:
        - name: helloworld
          image: docker-registry.xxx.com/hello_proxy
          imagePullPolicy: Always
          ports:
            - containerPort: 8080
          command: ["java"]
          args: ["-Dhttp.proxyHost=proxy-service", "-Dhttp.proxyPort=80", "-Dhttps.proxyHost=proxy-service", "-Dhttps.proxyPort=80", "-jar", "target/app.jar"]
```

这里我们将我们的服务通过`node.vks/intranet-ip: "true"`部署到了内网中。而hello_proxy的代码其实很简单

```java
@PostConstruct
private void init() {
  try {
      System.out.println("http.ProxyHost=" + System.getProperty("http.proxyHost"));
      System.out.println("http.ProxyPort=" + System.getProperty("http.proxyPort"));
      System.out.println("https.ProxyHost=" + System.getProperty("https.proxyHost"));
      System.out.println("https.ProxyPort=" + System.getProperty("https.proxyPort"));
      hostname = InetAddress.getLocalHost().getHostName();
  } catch (UnknownHostException e) {
      hostname = "unknown host";
  }
}

```

当然你可以自己写一个从外网下载文件的controller或者请求内网服务的controller来验证。
