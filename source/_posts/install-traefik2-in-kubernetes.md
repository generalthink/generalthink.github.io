---
title: 在kubernetes中安装traefik2
date: 2020-05-22 09:39:51
tags: [traefik2, Kubernetes]
---

### traefik2 DaemonSet

云原生微服务中我们使用了traefik2来作为我们的网关，当然我们也是通过DaemonSet(也可以使用deployment)的方式来部署到Kubernetes集群中。
DaemonSet部署之后的pod有如下特征
> Kubernetes集群中的每个work node上都有这个pod(traefik2实例)
> 每个work node上只有一个这样的pod实例
 当有新的work node加入Kubernetes集群后，该pod会自动在新加入的work node上被创建出来，而当旧的work node被删除后,它上面的pod也会被相应的删除掉。

<!--more-->

比如我们的traefik2的DaemonSet像下面这样

![DaemonSet](/images/k8s/traefik-ds.png)

需要注意的是args从configMap中引入了4个变量，其值分别如下:

```
# 访问http端口
web_port=80

# 访问https端口
websecure_port=443

# traefik dashborad端口(默认)
traefik_port=8080

# 监控哪个namespace资源,如果有多个,则以逗号连接。默认是所有namespace
watch_namespace=exmpale-beta
```

> 如果你使用DaemonSet的方式安装两个相同的traefik,那么上面提及到的4个变量都要进行对应的修改
> 一个traefik提供对内服务的访问，一个traefik提供对外服务的访问

使用了kubernetes,流量路由就如下所示

```
    internet
        |
   [ Ingress ]
   --|-----|--
   [ Services ]

```

这里定义的80/443端口就是外部的请求要通过这两个端口才能转发到Service(kubernetes)中。

#### 指定关注的的namespace

如果我们不指定traefik2监控的是哪个namespace的资源(默认是所有的),那么当我们访问的时候就可能访问到其他namespace的资源

```
curl -H 'Host: www.example-beta.com' www.example-rc.com
```

example-beta.com部署在example-beta这个namespace,而example-rc.com部署在example-rc这个namespace，但是通过上面的访问方式就能访问到example-beta的资源,很明显这并不安全。


![ingressRoute](/images/k8s/traefik-ingressroute-1.png)

traefik ingress解析规则是根据Host头来解析的，如果我们偷天换日就会导致访问了另外一个namespace的资源。
>关于traefik ingress route我在下面还会提及

### 使用traefik2 IngressRoute代替Kubernetes Ingress

traefik2不仅可以作为网关，它还可以作为ingress controller(另一个比较出名的的nginx),有了它我们需要使用ingress route来配置我们的匹配规则而不再使用kubernetes的ingress。


之前kubernetes 的ingress规则是这样的


![Ingress](/images/k8s/ingress.png)

而使用了ingress route之后则是这样的


![ingressRoute](/images/k8s/traefik-ingressroute-2.png)

### 为什么需要CRD?

IngressRoute这个Kind是traefik提供的,kubernetes并不认识它呀，不过还好kubernetes支持CRD(Custom Resource Definitions)。我们只需要加上CRD即可。

> 所有关于kubernetes的CRD配置请参考官方文档: https://docs.traefik.io/reference/dynamic-configuration/kubernetes-crd/


### 访问集群资源(RBAC)

这里着重需要说明的是RBAC: 基于角色的访问控制(Role-Based Access Control)的配置


![cluster-role](/images/k8s/traefik-cluster-role.png)


![cluster-role-binding-user](/images/k8s/traefik-cluster-role-binding-user.png)

RBAC实际上很简单,想想平时做的业务系统，你只要知道下面这几个概念你就知道了。

Kubernetes中的service,ingress，configmap,secret等都是资源,一般是一个账号，属于什么角色，这个角色可以操控哪些资源

> 比如HR可以查看员工工资,而员工只能知道自己的工资

所以Kubernetes中也存在着三个基本概念

1. Role : 角色，它其实是一组规则，定义了一组对Kubernetes API对象的操作权限
2. Subject：被作用者，既可以是“人”，也可以是“机器”，也可以使你在Kubernetes里定义的“用
户”。
3. RoleBinding：定义了“被作用者”和“角色”的绑定关系


![role-binding-user](/images/k8s/role-binding-user.png)


Role对象的rules字段，就是它所定义的权限规则。在上面的例子里，这条规则的含义就是：允许“被作用者”，对mynamespace下面的Pod对象，进行GET、WATCH和LIST操作。

RoleBinding对象里定义了一个subjects字段，即“被作用者”。它的类型是User，即Kubernetes里的用户。这个用户的名字是example-user。

这里的User实际上是Kubernetes的内置账户ServiceAccount。


在Kubernetes中Role和RoleBinding对象都是Namespaced对象，它们对权限的限制规则仅在它们自己的Namespace内有效。 对于非Namespaced（Non-namespaced）对象（比如：Node），或者，某一个Role想要作用于所有的Namespace的时候,就需要使用ClusterRole和ClusterRoleBinding这两个组合了。(也就是我们上面贴出来的配置)

至此,traefik2已经安装完毕。

### 总结

对应安装第三方提供的组件，安装是存在固定流程的。

1. 使用DaemonSet/Deployment的方式安装
2. 指定CRD
3. 指定RBAC

对于traefik2而已，如果你想要在work node上安装两个traefik(一个对内，一个对外),那么你至少需要改变三个端口。