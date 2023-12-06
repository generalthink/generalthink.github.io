---
title: k8s中如何进行DNS解析
date: 2023-11-23 09:53:16
tags: [Kubernetes, DNS]
---

之前文章<老大喊我7天从SpringCloud转到K8S>中我提到过我们使用open-feign来访问内部服务,有很多童鞋留言问我访问内部服务的url应该怎么写，但是还是整个系统的整体url吗？

比如我整个系统的url是  `http://www.think123.com`, 现在有user服务和manage服务,那我现在想要访问user服务的ip就变成 `http://www.think123.com/user`吗? 当然不是因为这样的话你的访问就相当于还是访问外网，又要重新走gateway, 这样子不仅增加网络开销，而且我们做鉴权也不方便，因为服务间的访问鉴权往往是通过一个注解或者token来实现的。

<!-more-->


### 服务的ip地址

我们都知道在k8s中我们一般会部署多个pod, 而我们又是通过K8S的Service来做LoadBalance的,Service会根据一定的策略来选择具体访问的pod, 如下图所示

![访问](/images/k8s/website-to-k8s.png)

因此实际上我们只需要知道service的地址就行了,使用下面的命令查看service的地址

```
kubectl get  svc -n <namespace>
```
![service](/images/k8s/svc-ip.png)

我们可以看到service的ip,因为我们服务是部署到同一个集群的，那是不是只需要知道服务的ip就行了，然后在open-feign中声明要访问的服务的ip加上端口就可以了呢？

原则是上可以的,但是实际上有问题,首先不同集群中ip地址不一样,如果你要部署不同集群，那么每次都要修改，其次就算只有一个集群，这个ip也是可能变动的，所以写ip不太靠谱。

### 服务的域名

好在kubernetes给某个服务都添加了一个域名(这是通过kube-proxy和iptables实现的), 域名的规则是  `..svc.cluster.local`。 

比如对于我们的user服务,我们这个地址就是 `user.dev.svc.cluster.local(<serviceName>.<namespace>.svc.cluster.local)`

当然后面的这么一堆你都可以省略，实际上我们在open-feign的url中只需要声明 `http://user:9201/(服务名:端口)` 就可以了,kubernetes在进行dns寻址的时候会先在本地dns找user这个域名对应的ip地址

### K8S的DNS解析机制

1. 集群域名和后缀：Kubernetes集群中的每个服务都会被分配一个域名，该域名由服务名称（Service Name）和命名空间（Namespace）组成。例如，一个服务名为my-service，位于命名空间my-namespace的服务的完整域名将是my-service.my-namespace.svc.cluster.local。svc.cluster.local是Kubernetes集群默认的后缀。

2. Kubernetes DNS服务器：Kubernetes集群内部有一个专用的DNS服务器负责处理服务的DNS解析请求。这个DNS服务器通常被命名为kube-dns或coredns。

3. 解析流程：在进行DNS解析时，应用程序或服务可以使用服务名作为主机名（hostname），然后发送DNS查询请求到Kubernetes DNS服务器。

4. DNS查询：Kubernetes DNS服务器接收到DNS查询请求后，会根据请求中的域名信息进行解析。它首先进行域名拆分，将服务名、命名空间和集群后缀分离开。

5. 域名解析：Kubernetes DNS服务器会依次解析域名的各个部分。它首先解析命名空间，然后根据服务名在该命名空间下查找对应的Service资源。

6. Service资源解析：Kubernetes DNS服务器在Service资源中查找与请求的服务名和命名空间匹配的条目。如果找到匹配项，将返回与之关联的Pod IP地址列表。

7. IP地址返回：Kubernetes DNS服务器将解析到的Pod IP地址返回给发起请求的应用程序或服务。

8. 重试机制：如果在初始查询时没有找到匹配的Service资源，Kubernetes DNS服务器可能会进行一些重试机制，以确保服务名得到正确解析。这样做是因为在创建和删除Service资源的过程中，可能会存在一定的延迟。
通过这种方式，Kubernetes DNS解析机制使得服务能够通过服务名进行通信，无需关心具体的Pod IP地址。这种抽象层简化了服务之间的通信配置，并支持动态扩展和管理服务。


#### 查看域名

可以通过下面的命令查看域名

```
kubectl get svc my-service -n my-namespace -o jsonpath='{.metadata.name}.{.metadata.namespace}.svc.cluster.local'
```

#### 查看coredns

通过下面的命令可以查看coredns pod

```
kubectl -n kube-system get pods -l k8s-app=kube-dns
```

当然我们可以通过下面的命令查看coredns的配置

```
kubectl -n kube-system get cm -l k8s-app=kube-dns
kubectl describe cm coredns -n kube-system
```

其核心配置如下

```
.:53 {
    errors
    ready
    health
    kubernetes cluster.local in-addr.arpa ip6.arpa {
      pods insecure
      fallthrough in-addr.arpa ip6.arpa
    }
    prometheus :9153
    forward . /etc/resolv.conf
    cache 30
    loop
    reload
    loadbalance
    import custom/*.override
}


```
1. `.:53` 表示监听的 DNS 端口号为 53，. 表示根域名（Root Zone）。

2. `kubernetes cluster.local in-addr.arpa ip6.arpa` 定义了多个域和反向解析配置项。

	+ `kubernetes` 是 Kubernetes 插件的名称，用于解析 Kubernetes 集群的服务和 Pod。

	+ `cluster.local` 是 Kubernetes 集群内部域名的默认后缀。

	+ `in-addr.arpa` 和 `ip6.arpa` 是用于反向 DNS 解析的 IPv4 和 IPv6 地址后缀。

	+ `pods insecure` 允许对 Pod 进行非安全（insecure）的 DNS 解析。

	+ fallthrough in-addr.arpa ip6.arpa 表示如果查询未匹配到任何资源记录，则继续向下查询反向 DNS 解析。
3. `forward . /etc/resolv.conf` 将未能解析的 DNS 请求转发给 /etc/resolv.conf 文件中配置的其他 DNS 服务器。



大家也可以去看pod中中/etc/host和/etc/resolv.conf然后也能发现一些迹象。


### 结束语

至此,讲清楚了集群中内部服务之间是如何访问的，且k8s是如何进行寻址导致对应的pod的,希望对大家有所帮助。