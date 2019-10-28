---
title: 让你的SpringBoot一次build,到处运行
date: 2019-10-25 13:54:53
tags:
  - spring
  - k8s configMap
---

开发web项目的时候，我们一般会有多个环境(dev,beta,rc,production),然后每次使用maven打包的时候都要指定profile

```
mven clean package -P beta
```
这样就可以将application-{profile}.properties打入jar包中。这也就意味着对于不同的环境我们要打不同的包，虽然他们只有一些配置信息不同而已。

那么有没有办法在不同的环境中使用相同的应用程序代码呢? 当然有,[springboot提供了这样的机会](https://docs.spring.io/spring-boot/docs/current/reference/html/spring-boot-features.html#boot-features-external-config),它允许我们通过很多种方式来决定配置文件是哪一个,当然啦本篇文章并不是介绍有哪些加载外部配置文件的方式。

<!--more-->

就比如祸水三千，我只取一瓢。所以这次我们直接指定property文件的位置即可

```
java -jar myproject.jar --spring.config.location=classpath:/default.properties,/myconfig/application.properties
```

道理是这么个道理,但是现在大多的项目都是部署在k8s上的,那么应该如何做呢？

这里我借助了k8s configMap来实现加载外部配置文件达到我们的目的。

首先这里生成了configMap
```
kubectl create configmap application-configmap --from-file=application.properties=../smcp-web/src/main/resources/application-beta.properties -o yaml -n smcp| kubectl replace -f -
```

解释下上面命令的意思,每一次在打包镜像之后，都会重新生成configMap,生成的application-configmap中，key=application.properties,value则是项目中application-beta.properties的内容
> 生成configmap有很多种方式，可以根据自己的需求来生成，可以查看官方文档 https://kubernetes.io/docs/tasks/configure-pod-container/configure-pod-configmap/
> -n 的意思是指定namespace,如果你使用的是默认的namespace,那么可以不用指定。 -n == --namespace

我们生成的上面的configMap可以通过` kubectl get configmap application-configmap -o yaml -n smcp`查看

```yaml
apiVersion: v1
data:
  application.properties: |
    logging.level.com.project.smcp=INFO
    logging.level.org.apache.shiro=INFO
    logging.level.okhttp3.OkHttpClient=ERROR
kind: ConfigMap
metadata:
  creationTimestamp: 2019-10-24T11:41:57Z
  name: application-configmap
  namespace: smcp
  resourceVersion: "62755048"
  uid: 48c8ab36-f653-11e9-9eab-fa163fea9021

```
可以看到其实是将application-beta.properties文件中的内容全部拷贝到了data中，然后这些所有值的key为application.properties.再来看看在我们deployment.yml文件中如何使用

```yaml
apiVersion: apps/v1beta2
kind: Deployment
metadata:
  name: smcp-service
  namespace: smcp
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
        image: docker-registry.xxx.com/xxx/smcp-web:latest
        args: ["--spring.config.location=/smcp-config/application.properties"]
        ports:
        - containerPort: 9000
        volumeMounts:
        - name: application-config
          mountPath: smcp-config
        envFrom:
        - secretRef:
            name: my-secret
      volumes:
      - name: application-config
        configMap:
          name: application-configmap
          items:
            - key: application.properties
              path: application.properties
---
apiVersion: v1
kind: Service
metadata:
  name: smcp-service
  namespace: smcp
  labels:
    app: smcp-service
spec:
  ports:
  - targetPort: 9000
    port: 9000
    protocol: TCP
  selector:
    app: smcp-service
  type: NodePort
```

经过上面的配置，现在application.properties文件就处于/smcp-config下了。看到这里可能会有人有疑问,为什么我的configMap要单独用命令生成，而不是在demployment.yaml文件中单独声明configMap类型的配置呢?
其实也是可以的，但是如果这样做你每次添加了环境变量你都要去demployment.yml中添加配置,你这样可能就要配置两次(application-{profile}.properties中还要配置一次)，所以我就干脆用命令生成，在开发的时候不要考虑k8s的存在。

还要注意上面的args参数，通过它我们指定了外部环境变量的路径。而对于生成镜像的dockerfile，其实也很简单
```
FROM java:8
EXPOSE 9090
ADD target/smcp-web.jar /smcp-web.jar
ENTRYPOINT ["java", "-jar","/smcp-web.jar"]
```

最后当我们执行了下面的命令之后，就可以部署我们的容器到k8s中了

```
kubectl apply -f deployment.yml
```

最后部署后，进入Pod中，可以看到目录结构如下

```
smcp-web.jar
smcp-config
  -- application.properties
```

而启动命令也变成了下面这样，可以通过`ps -ef`查看

```
java -jar /smcp-web.jar --spring.config.location=/smcp-config/application.properties
```

上面使用configMap的方式处理,当然还可以直接在环境变量中使用configMap(参考上面secretRef的方式)，当使用环境变量的方式注入configMap的时候，你需要使用下面这样的命令生成数据。

```
kubectl create configmap application-configmap --fron-env-file=src/main/resources/application-beta.properties
```

同时yaml文件中关于configMap的修改成下面这样,使用环境变量的方式记得把volumes去掉。
```
envFrom:
- configMapRef:
  name: application-configmap
```
但是使用环境变量注入的方式有一点需要注意,当你更新了configMap而不重新部署的时候,容器中的变量是不会更新的，而如果使用mountPath的方式,环境变量的值就会更新(大概10s左右)。
>如果你还使用mountPath的同时还使用了subpath同样不会更新

