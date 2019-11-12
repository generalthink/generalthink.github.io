---
title: K8S集群中安装Nginx Ingress Controller
date: 2019-11-08 14:45:10
tags:
- nginx-ingress
- ingress
- kubernetes
- k8s
---

Kubernetes Ingress是为了代理不同后端Service而设置的负载均衡服务,我们自己的每个服务可以根据自己的需求定制自己的ingress rule.


实际的使用中我们要选择一个具体的ingress controller，部署在k8s集群中，然后Ingress Controller会根据我们定义的Ingress对象,提供对应的代理能力。

比如我在们开发一个cms系统,是前后端分离的，也是分开部署的，我希望它们有这样的规则

```
http://www.mycms.com/api   ---> 后端

http://www.mycms.com/   --->  前端

```

<!--more-->

ingress很简单，下面这样就可以

```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: cms-ingress
  namespace: today-partner-beta
  annotations:
    nginx.ingress.kubernetes.io/proxy-body-size: "20M"
spec:
  tls:
  - hosts:
    - mycms.com
    secretName: cms-secret
rules:
- host: mycms.com
  http:
    paths:
    - path: /api
    backend:
      serviceName: cms-service
      servicePort: 9000
    - path: /
    backend:
      serviceName: cms-html
      servicePort: 80
```

当我们创建了ingress之后，可以使用下面的命令查看
```
$ kubectl get ingress cms-ingress -n today-partner-beta
NAME           HOSTS       ADDRESS          PORTS     AGE
cms-ingress   mycms.com   172.19.88.130   80, 443     4d
```

上面ingress rule这样就会形成下面的映射
```

mycms.com -> 172.19.88.130 -> /api  cms-service:9000
                              /     cms-html:80

```


当然在这个ingress生效之前，我们首先要在集群中安装ingress controller,它会收集所有的ingress rule,然后在遇到请求的时候进行转发

这里我们选择的是[nginx ingress controller](https://github.com/kubernetes/ingress-nginx/blob/master/docs/deploy/index.md),安装步骤如下

1. 执行下面的命令

```
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/static/mandatory.yaml
```
如果你使用的是1.14之前的Kubernetes版本，则需要在mandatory.yaml的第217行将kubernetes.io/os更改为beta.kubernetes.io/os，


2. 使用nodeport的方式暴露nginx-controller

```
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/static/provider/baremetal/service-nodeport.yaml
```

3. 验证安装是否成功

```
kubectl get pods --all-namespaces -l app.kubernetes.io/name=ingress-nginx --watch
```
如果你看到ingress controller的状态是running，就代表安装成功，现在你可以安装自己的ingress了。


4. 查看nginx ingress controller暴露的端口

```
$ kubectl get svc -n ingress-nginx
NAME            TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)                      AGE
ingress-nginx   NodePort   172.19.88.130   <none>        80:32257/TCP,443:30839/TCP   6d

```
可以看到这里对外暴露出来的端口是32257和30839两个。
>这两个端口是在你配置LoadBalance的时候要用的,80-->32257,443-->30839

我们的nginx-ingress-controller部署好之后，它会去拉取集群中的所有ingress-rule,当请求通过worker node转发到ingress controller之后，它会根据具体的规则将请求转发到对应的Service.

**尤其需要注意的是我们k8s集群的网络是私有的，如果你想要访问集群内的服务，只能通过worker node ip才能访问对应的服务。**

比如我的k8s集群中有3个worker node,它们的ip分别是下面这样

```
10.231.123.142
10.231.123.143
10.231.123.147

```

一般我们会添加一个LoadBalancer,映射这三个workerer node,这个时候会分配给你一个VIP,然后在DNS中设置域名到VIP的映射


```
mycms.com   10.128.161.21
```

>DNS也是可以实现负载均衡的，只是DNS的实现和专业的LoadBanlance相比还是有很大区别的


![Load Balancer](/images/k8s/lb-http.png)
![Load Balancer](/images/k8s/lb-https.png)

这样当我们访问https://mycms.com/api的时候,请求通过LB转发到worker node,然后通过ingress controller转发到具体的service了。


如果你是在windows上，你可以配置hosts文件，mycms.com指向worknode ip即可。


** 413 Request Entity Too Large**

当文件上传的时候你遇到这个错误，就证明你需要适当调整你的ingress配置了

```
nginx.ingress.kubernetes.io/proxy-body-size : "20M"
```

