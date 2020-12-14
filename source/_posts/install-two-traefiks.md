---
title: 架构师喊我安装两个traefik
date: 2020-12-10 17:10:48
tags: [traefik, kubernetes]
---

之前有说过，我用traefik做网关，无论是内外网请求都会经过网关。


![请求](/images/k8s/request-path.png)

但是我们有一部分API是只有内网会用，为了安全，我们要保证这些内网的API只有内网可以访问到。

但是由于之前的设置，这些API是匿名访问的，如果修改为需要权限，那么需要其他依赖于我们服务的team来做对应的修改，是由于一些原因，这条路走不通。

<!--more-->

摆在我面前的就只有一条路，那就是安装两个traefik,一个对内，一个对外。安装两个traefik很容易，由于traefik安装采用的是DaemonSet的方式，所以两个traefik的访问端口必须不同

```
--entryPoints.traefik.address=:8080
```

比如internet(外网)的traefik的端口是8080，那么internal的traefik就可以是9090。

traefik安装请看我之前的文章--[在 kubernetes 中安装 traefik2](https://generalthink.github.io/2020/05/22/install-traefik2-in-kubernetes/)。

然后就可以啦？ 当然没有那么容易,为了实现下面的效果，

![请求](/images/k8s/request-path-2.png)

对应的IngressRoute也同样需要修改

以前的IngressRoute如下

```yaml
apiVersion: traefik.containo.us/v1alpha1
kind: IngressRoute
metadata:
  name: my-service-traefik-ingress
spec:
  entryPoints:
    - websecure
  routes:
    - match: Host(`think123.com`) && PathPrefix(`/api/anon/health`, `/api/anon/article`)
      kind: Rule
      services:
        - name: my-service
          port: 9000
```

我想要`/api/anon/article`这个request path只能内网访问，那么需要将配置修改如下

```yaml
apiVersion: traefik.containo.us/v1alpha1
kind: IngressRoute
metadata:
  name: my-service-traefik-ingress
spec:
  entryPoints:
    - websecure
  routes:
    - match: Host(`think123.com`) && PathPrefix(`/api/anon/health`)
      kind: Rule
      services:
        - name: my-service
          port: 9000

apiVersion: traefik.containo.us/v1alpha1
kind: IngressRoute
metadata:
  name: my-service-internal-traefik-ingress
spec:
  entryPoints:
    - websecure
  routes:
    - match: Host(`think123-internal.com`) && PathPrefix(`/api/anon/article`)
      kind: Rule
      services:
        - name: my-service
          port: 9000

```

看上去没问题了，配置好LoadBalance之后，访问对应的域名就可以了。 但是这样还存在一个问题

```
curl -H 'Host:www.think123-internal.com' -X GET https://www.think123.com/api/anon/article 
```

通过指定header的方式，我们发现它匹配了 `my-service-internal-traefik-ingress`这个ingressRoute，最终返回了只有内网用户才能访问的API。

为啥我做了一大堆，安全性问题仍然存在？

实际上是因为无论是内网的traefik还是只监控外网的traefik，它们两个都会加载所有的IngressRoute,所以就有了上面的那个问题。

必须想个办法让它们只加载属于自己的IngressRoute,我去翻阅了下Traefik的文档，发现有3个参数可以用。

第一个是可以指定要监视哪些namespaces,则traefik就只处理它监控下的namespaces请求

```
--providers.kubernetescrd.namespaces=default,production
```

第二个是给资源打上label,但是只对Traefik Custom Resources起作用，对Kubernetes的Service,Secrets这些不起作用

```
--providers.kubernetescrd.labelselector="app=traefik"
```

第三个是指定ingressClass

```
--providers.kubernetescrd.ingressclass=traefik-internal
```
我们需要在对应的资源上加上kubernetes.io/ingress.class注解，用于标识要处理的资源对象。


由于我们处理的是同一个namespace下的资源，所以namespace的方式不合适。

而对于第二种或者第三种而言，由于我们对IngressRoute进行处理，所以都是合适的。但是考虑到其他环境存在nginx和traefik共存的情况，我们决定使用ingressClass的方式,实际上nginx也是建议我们这么做的。

![multi ingress controller](/images/k8s/multi-ingress-controller.png)


最开始我的修改是这样的,首先在DaemonSet模板中添加ingressClass设定

```
--providers.kubernetescrd.ingressclass=$(ingress_class)
```

接下来只需要在不同的traefik中配置不同的环境变量(configMap)即可。比如internet traefik设置`ingress_class=traefik-internet`,而intranet traefik该值为空,它被当成默认的ingress controller。

然后在相应的ingressRoute中指定 `kubernetes.io/ingress.class: traefik-internet`就可以了。 其他IngressRoute不指定该注解。

看上去很棒，给自己点个赞。

可是想象是美好的，现实却给了我无情一击。这样修改完成之后，我发现当我访问 `https://www.think123-internal.com/api/anon/article`这个api的时候,没有traefik来处理我的请求。

internal traefik不是我的默认traefik吗？我看官方文档还这样写的呀

```
If the parameter is non-empty, only resources containing an annotation with the same value are processed.
Otherwise, resources missing the annotation, having an empty value, or the value traefik are processed
```

难道这个空不是空串，而是不指定的意思？ 真是不讲武德！


所以，我只能换一种修改方式，在DaemonSet的模板中不指定ingressClass,而是在通过kustomize来动态添加。kustomize的使用请参考我的[这篇文章](https://generalthink.github.io/2020/01/03/use-kustomize/)

> 注意，两个traefik只有些许配置不一样，所以，我们安装的时候一般指定一个模板，动态的值则由不同的traefik来指定

```yaml
patches:
  - target:
      version: v1beta2
      kind: DaemonSet
      name: traefik
    patch: |-
      - op: add
        path: /spec/template/spec/containers/0/args/-
        value: --providers.kubernetescrd.ingressclass=traefik-internet
```

> 在我的配置中，internet traefik名字叫做traefik,internal traefik名字叫做traefik-internal,区分内外网是逻辑区分，而不仅仅是通过名字。

我们只在internet traefik中指定ingressClass,intranet traefik则不指定。

同样的，IngressRoute也需要动态修改的。

```yaml
# kustomization.yaml
patchesJson6902:
- target:
    group: traefik.containo.us
    version: v1alpha1
    kind: IngressRoute
    name: my-service-traefik-ingress
  path: patches/patch-internet-ingressroute.yaml

# patch-internet-ingressroute.yaml
- op: add
  path: /metadata/annotations
  value:
    kubernetes.io/ingress.class: traefik-internet
```

这样，internet traefik就只会处理 `kubernetes.io/ingress.class: traefik-internet`的ingressRoute了。之前的安全性问题也就不存在了。

### 参考文档

1. https://kubernetes.github.io/ingress-nginx/user-guide/multiple-ingress/
2. https://doc.traefik.io/traefik/providers/kubernetes-crd/#ingressclass