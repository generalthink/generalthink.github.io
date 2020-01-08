---
title: 使用kustomize来管理Kubernetes yaml应用
date: 2020-01-03 16:05:32
tags:
- Kubernetes
- Kustomize
---


Kustomize是为了解决k8s yaml应用管理问题而生的一个工具，在1.14版本之后kubectl就集成了kustomize,而在这之前，我们则只能自己安装。
可以在github上下载对应操作系统的包进行安装(https://github.com/kubernetes-sigs/kustomize/releases)。

windows下它就是一个exe文件，我们把它放到某一个目录后,加入环境变量，即可在命令行中进行使用了。

```
$ kustomize version
{Version:kustomize/v3.5.2 GitCommit:79a891f4881cfc780e77789a1d240d8f4bfa2598 BuildDate:2019-12-17T03:48:17Z GoOs:windows GoArch:amd64}
```
<!--more-->

好了，重新回到k8s,当我们部署的时候，是不是会使用很多kubernetes 对象来描述我们的服务。比如deployment,ingress,service,configMap,secret等等。这些我们都是使用yaml文件来进行配置的。比如一个典型的deployment.yaml文件长这样子

```yaml
apiVersion: apps/v1beta2
kind: Deployment
metadata:
  name: smcp-service
  namespace: smcp-dev
spec:
  selector:
    matchLabels:
      app: smcp-service
  replicas: 1
  template:
    metadata:
      labels:
        app: smcp-service
    spec:
      containers:
        - name: smcp-service
          image: harbor.xxx.com/example/smcp-web:1.0
          imagePullPolicy: Always
          livenessProbe:
            httpGet:
              path: /api/anon/health
              port: 9000
            initialDelaySeconds: 80
            periodSeconds: 20
            timeoutSeconds: 2
          readinessProbe:
            httpGet:
              path: /api/anon/health
              port: 9000
            initialDelaySeconds: 90
            timeoutSeconds: 2
          ports:
            - containerPort: 9000
          envFrom:
            - configMapRef:
                name: smcp-config
            - secretRef:
                name: smcp-service-secret

```

比如上面这个springboot应用,我们部署到dev环境，可以看到namespace是smcp-dev,但是当我们部署到其他环境,比如beta,staging,production环境，意味着我们要把上面所说的这些文件都抄一遍，然后修改其中的微小变量(namespace,replica,image等)。

而且当我们环境变量的内容如果被更新了，这个deployment也不会滚动更新,因为它们的name没有变化(smcp-config,smcp-service-secret)。
>我们将application.properties文件的内容通过configMap放到环境变量中了。


而kustomize应运而生，它可以解决我们的问题。它将公共的部分提出来作为base,即基础层,然后在overlays上对base中的内容进行覆盖,有点类似docker image layer的概念。而base对overlays是无感知的。最终形成的目录结构是这样的

```
$ tree
.
|-- base
|   |-- deployment.yaml
|   |-- ingress.yaml
|   |-- kustomization.yaml
|   `-- service.yaml
`-- overlays
    |-- devlopment
    |   |-- cert
    |   |   |-- tls.crt
    |   |   `-- tls.key
    |   |-- config.properties
    |   |-- ingress-patch.yaml
    |   |-- kustomization.yaml
    |   `-- secret.properties
    `-- production
        |-- cert
        |   |-- tls.crt
        |   `-- tls.key
        |-- config.properties
        |-- ingress-patch.yaml
        |-- kustomization.yaml
        `-- secret.properties

6 directories, 16 files
```
而base/kustomization.yaml文件中只是引入了其他几个文件,它的yaml文件内容如下

```yaml
resources:
- deployment.yaml
- ingress.yaml
- service.yaml
```

base中比如ingress.yaml或者service.yaml文件只是定义了deployment对象以及service对象，对于某些需要根据不同的环境来设定的参数，比如namespace,replicas等，你可以先设置一个默认值，然后在overlays中修改就行了。

>kustomize create可以生成kustomization.yaml文件


而在overlays可以看到我们在overlays文件夹下放置了两个环境的配置,分别存储的是这两个环境中独有的配置(config.properties,secret.properteis),而kustomizatiion.yaml文件中这是我们对模板文件(base)内容的一些修改,我们来看看development环境下的配置

```yaml
resources:
- ../../base

# 设置namespace
namespace: smcp-example-dev


replicas:
- name: smcp-service
  count: 2

# 设置configMap
configMapGenerator:
- name: smcp-config
  files:
  - config.properties

# 设置secret
secretGenerator:
- name: smcp-service-secret
  envs:
  - secret.properties
- name: smcp-service-cert
  files:
  - cert/tls.crt
  - cert/tls.key
  type: kubernetes.io/tls

# 这里的group/version的值等于ingress.yaml文件中apiVersion的值
patchesJson6902:
- target:
    group: extensions
    version: v1beta1
    kind: Ingress
    name: smcp-service-ingress
  path: ingress-patch.yaml

# 设置镜像版本
images:
- name: harbor.xxx.com/example/smcp-web
  newTag: 0.10.0-test
```

而ingress-patch.yaml文件中内容如下,实际上是修改ingress中关于域名的配置,将原有的域名修改为​dev环境特有的。​

```yaml
- op: replace
  path: /spec/tls/0/hosts/0
  value: test.example.com

- op: replace
  path: /spec/rules/0/host
  value: test.example.com

```

> kustomize的文档地址: https://github.com/kubernetes-sigs/kustomize/tree/master/docs

最后当我们执行` kustomize build overlays/devlopment `的时候新生成的api对象(deployment,ingress等),就会按照我们设置的那样将模板中的值进行替换。

新生成的Deployment对象如下

```yaml
apiVersion: apps/v1beta2
kind: Deployment
metadata:
  name: smcp-service
  namespace: smcp-example-dev
spec:
  replicas: 2
  selector:
    matchLabels:
      app: smcp-service
  template:
    metadata:
      labels:
        app: smcp-service
    spec:
      containers:
      - envFrom:
        - configMapRef:
            name: smcp-config-f5thmfmkb6
        - secretRef:
            name: smcp-service-secret-66m2658btb
        image: harbor.xxx.com/example/smcp-web:0.10.0-test
        imagePullPolicy: Always
        livenessProbe:
          httpGet:
            path: /api/anon/health
            port: 9000
          initialDelaySeconds: 80
          periodSeconds: 20
          timeoutSeconds: 2
        name: smcp-service
        ports:
        - containerPort: 9000
        readinessProbe:
          httpGet:
            path: /api/anon/health
            port: 9000
          initialDelaySeconds: 90
          timeoutSeconds: 2
```
可以看到相比较于base中的deployment.yaml,
replicas变成了2,namespace变成了smcp-example-dev,configMap和secret的生成规则这是`${name}-hash`,一切都是按照kustomization.yaml文件的配置来进行修改的。

而且现在只要我们更新了configMap和secret,也就能触发滚动更新了(**deployment中只有spec.spec中的内容更新了才会触发滚动更新**)。
> 可以配合kubectl命令使用: kustomize build overlays/devlopment | kubectl apply -f -

至此，我们就完成了kustomization的使用了，有了它，我们可以很快部署一套系统到其他环境，同时修改基础配置，也只用修改一次，而不是在不同的环境都需要修改了。

