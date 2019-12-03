---
title: K8S中只挂载文件,不要挂载目录
date: 2019-11-19 15:04:19
tags:
- K8S
- Kubernetes
---

在kubernetes中部署前端项目(使用nginx作为服务器)的时候,遇到了一个报错,报错信息如下

```
2019/11/19 02:16:31 [emerg] 1#1: open() "/etc/nginx/mime.types" failed (2: No such file or directory) in /etc/nginx/nginx.conf:14
nginx: [emerg] open() "/etc/nginx/mime.types" failed (2: No such file or directory) in /etc/nginx/nginx.conf:14
```

提示的意思是没有找到mine.types文件,也就是说容器内/etc/nginx下不存在这个文件。但是这个文件不是nginx提供的吗?

<!--more-->

我赶忙看了下我是如何部署这个容器的。

dockerfile:

```
FROM nginx:1.17.3
COPY dist /usr/share/nginx/html
```

deployment.yml:

```yaml
apiVersion: apps/v1beta2
kind: Deployment
metadata:
  name: smcp-web
  namespace: nginx-test
spec:
  selector:
    matchLabels:
      app: smcp-web
  replicas: 1
  template:
    metadata:
      labels:
        app: smcp-web
    spec:
      containers:
      - name: smcp-web
        image: docker-registry.xxx.com/fe/fe-nginx:no-nginx-conf
        ports:
        - containerPort: 80
        volumeMounts:
        - name: nginx-conf
          mountPath: /etc/nginx
      volumes:
      - name: nginx-conf
        configMap:
          name: nginx-conf
          items:
            - key: nginx.conf
              path: nginx.conf

```

这样就会将在configMap中声明的nginx.conf对应内容挂载到/etc/nginx(mountPath)/nginx.conf文件中。
> kubectl create configmap nginx-conf --from-file=nginx.conf=./path/to/nginx.conf -n nginx-test

一想到这里我大概知道了原因，**是因为我们挂载的是整个/etc/nginx目录,但是我们这个目录里面只加了nginx.conf,而容器中/etc/nginx目录下面还有很多其他的文件，
我们直接挂载了整个目录进去，其他的文件会随着目录的挂载而消失，自然读取配置就出了问题。**

现在问题已经知道了，如何解决呢?

要是可以只挂载这个文件就好了，查阅了下文档，发现subPath完全可以满足我们的需求。
>subPath的目的是为了在单一Pod中多次使用同一个volume而设计的

所以将spec部分修改如下:

```yaml
spec:
  containers:
  - name: smcp-web
    image: docker-registry.xxx.com/fe/fe-nginx:no-nginx-conf
    ports:
    - containerPort: 80
    volumeMounts:
    - name: nginx-conf
      mountPath: /etc/nginx/nginx.conf
      subPath: nginx.conf
  volumes:
  - name: nginx-conf
    configMap:
      name: nginx-conf
```

这样我们就做到了将nginx.conf文件挂载到/etc/nginx目录下，同时不影响原有目录中的内容。

