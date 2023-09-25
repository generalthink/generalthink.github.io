---
title: from-spring-cloud-to-springboot
date: 2023-09-14 10:20:10
tags: [springcloud,springboot,k8s]
---


之前项目使用的是springcloud,主要使用到的组件有 spring gateway, nacos, minio, load balancer,open-feign等,然后我们的微服务通过docker部署到虚拟机里面的。

但是出于安全的考虑,需要将它迁移到azure的aks(kubernetes)中,所以需要将spring cloud改造成spring boot。这样就不用自己维护虚拟机的安全策略，也不需要去关注补丁了。


### 梳理项目结构

项目是一个一个微服务组织起来的，大概业务类的服务有5个,公共服务有4个。 设计到的改造主要集中在gateway, auth中，公共包的一些改造比较少，主要是将open-feign的访问改为通过url进行调用，而不是之前通过服务名来。

而在kubernetes中，我们使用Traefik2来代替gateway的功能，不知道traefik2的，可以去翻翻之前的文章。

同时对于授权,需要提供一个授权接口，配合traefik2使用，这样每一个请求都会进行授权的验证。

### 开始改造

#### 确定分支

最开始肯定是新拉一个分支进行这些改动，即便没改好也不影响其他人，所以我们先把分支名定好就叫做 feature/AKS-migrate。

#### 改造gateway

首先把pom文件中的不需要的依赖包注释掉,比如spring cloud gateay, nacos, sentinel等spring cloud相关组件。注释掉之后
去看代码中有哪些报错，就针对性的修改。


我们的项目中使用蛮多gateway的filter以及handler,最开始我看他们都是使用的webflux,我就想我单独引入这个包，代码是不是最小的改动就行了呢？

这样尝试过之后我发现不行,因为项目中使用最多还是@RestController,如果使用webflux的方式,那么很多filter不生效的。 

所以这种方式也不行，但是代码改得太多了，我只好回退到注释pom文件依赖的那一步。

没办法,只能读之前对应的代码逻辑,然后将其转换了。


读取gateway filter的代码,将其转换成spring filter,直接继承 `org.springframework.web.filter.OncePerRequestFilter` 即可，然后将之前的逻辑搬过来。 

需要注意的是如果是全局filter需要放到公共包里面。

handler也是一样的,将其转换成filter,需要注意执行顺序。

这样,核心代码改造完毕,可以开启调试了

#### 遇到的坑

处理上面说的webflux的问题外,将springcloud变成springboot后,我们之前的配置文件名称是bootstrap.yml,bootstrap-dev.yml文件,但是改成了springboot后，配置文件名要改成application.yml,application-env.yml。

不然你会发现你启动不了,说找不到文件，这个坑也是自己把自己给坑了。



#### 改造nacos

前面提到了nacos主要在open-feign的调用中以及变量注入中使用到了。feign那个好改,只需要指定url参数即可,这样就可以去掉nacos的依赖了。 然后变量注入同样的我们可以使用Kubernetes的ConfigMap以及Secret来代替。 

所以我们需要将以前配置到nacos中的变量放到配置文件中,这样变量可以直接通过Kubernetes进行注入了。

我们在各个环境中只需要有一份代码(一个镜像),部署的时候只需要注入的配置不一样就可以了，这样就可以保证各个环境代码一致。

比如之前的配置是这样的

```yml
spring:
  redis:
    host: 127.0.0.1
    port: 6379
    database: 0
    password: 123456

```

改造后配置文件的值为

```yml
spring:
  redis:
    host: ${env_redis_host}
    port: ${env_redis_port}
    database: ${env_redis_database}
    password: ${env_redis_password}
```

而这里的变量是通过ConfigMap进行配置的,到时候会注入到容器环境变量中，这样spring就可以从环境变量中获取到值了。


### 部署

之前使用的jenkins方式部署，jenkins也是自己搭建的，现在全部迁移到了azure github上,所以这里直接使用azure的pipeline进行部署。 而我们管理k8s的资源则使用的是helm。


比如我项目中使用helm生成后结构如下

```
C:.                 
│  .helmignore      
│  Chart.yaml       
│  values-prod.yaml 
│  values-qa.yaml   
│  values-test.yaml 
│  values.yaml      
│                   
├─charts            
├─config            
│  ├─dev            
│  │      config.yaml
│  │      secret.yaml
│  │                
│  ├─prod           
│  │      config.yaml
│  │      secret.yaml
│  │                
│  ├─qa             
│  │      config.yaml
│  │      secret.yaml
│  │                
│  └─test           
│          config.yaml
│          secret.yaml
│                   
└─templates         
        configmap.yaml
        deployment.yaml
        hpa.yaml    
        secret.yaml 
        service.yaml
        _helpers.tpl

```

这里只需要部署的时候指定不同的value 文件，就可以实现同一个镜像部署到不同的环境了。

dev目录下config.yaml，secret.yaml文件内容大致如下:

```yml
# config.yaml
env_redis_host: localhost
env_redis_port: 6379
env_redis_database: 1


#secret.yaml

env_redis_password: 123456

```
在template中configmap.yaml, secret.yaml中主要是如何将文件内容转换成对应的yaml

```yml
#values.yaml 指定有哪些文件
configOverrides:
  - config/dev/config.yaml
secretOverrides:
  - config/dev/secret.yaml



# configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ include "bv-manifesto.fullname" . }}-configmap
  namespace: {{ .Values.nameSpace }}
data:
{{- $files := .Files }}
{{- range  .Values.configOverrides }}
{{- range $key, $value :=  ($files.Get (printf "%s" .) | fromYaml) }}
{{ $key | indent 2 }}: {{ $value | quote }}
{{- end }}
{{- end }}

# secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: {{ include "bv-manifesto.fullname" . }}-secret
  namespace: {{ .Values.nameSpace }}
type: Opaque
data:
{{- $files := .Files }}
{{- range  .Values.secretOverrides }}
{{- range $key, $value :=  ($files.Get (printf "%s" .) | fromYaml) }}
{{ $key | indent 2 }}: {{ $value | b64enc }}
{{- end }}
{{- end }}


```

最后给我deployment.yaml的例子,大家可以参考下

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "bv-manifesto.fullname" . }}
  namespace: {{ .Values.nameSpace }}
  labels:
    {{- include "bv-manifesto.labels" . | nindent 4 }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      {{- include "bv-manifesto.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      labels:
        {{- include "bv-manifesto.selectorLabels" . | nindent 8 }}
      annotations:
        {{- if .Values.configOverrides}}
        checksum/config: {{ include (print $.Template.BasePath "/configmap.yaml") . | sha256sum }}
        {{- end }}
        {{- if .Values.secretOverrides}}
        checksum/secret: {{ include (print $.Template.BasePath "/secret.yaml") . | sha256sum }}
        {{- end }}
    spec:
      {{- with .Values.imagePullSecrets }}
      imagePullSecrets:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      containers:
        - name: {{ .Chart.Name }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          ports:
            - name: {{ .Values.service.portName }}
              containerPort: {{ .Values.service.port }}
          envFrom:
            {{- if .Values.configOverrides }}
            - configMapRef:
                name: {{ include "bv-manifesto.fullname" . }}-configmap
            {{- end }}
            {{- if .Values.secretOverrides }}
            - secretRef:
                name: {{ include "bv-manifesto.fullname" . }}-secret
            {{- end }}
          {{- with .Values.livenessProbe }}
          livenessProbe:
            {{- toYaml . | nindent 12 }}
          {{- end }}
          {{- with .Values.readinessProbe }}
          readinessProbe:
            {{- toYaml . | nindent 12 }}
          {{- end }}
          resources:
            {{- toYaml .Values.resources | nindent 12 }}
      {{- with .Values.nodeSelector }}
      nodeSelector:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      {{- with .Values.affinity }}
      affinity:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      {{- with .Values.tolerations }}
      tolerations:
        {{- toYaml . | nindent 8 }}
      {{- end }}

```

至于values.yaml我就不给出来了，基本上是其他模板需要什么就写在上面就好了。

和Kustomize相比,helm安装第三方chart很方便，它有自己的仓库，这里附上我安装traefik2的命令

```
# 添加traefiK仓库
helm repo add traefik   https://traefik.github.io/charts    
#添加国内仓库       
helm repo add stable http://mirror.azure.cn/kubernetes/charts       
helm repo add aliyun https://kubernetes.oss-cn-hangzhou.aliyuncs.com/charts       
helm repo update       
helm repo list

helm install --set deployment.kind=DaemonSet --set namespaceOverride=traefik --set service.enabled=false traefik traefik/traefik

```

#### 本地验证

当你写好了上面的chart之后,如果你本地没有kubernetes环境(因为它可能在服务器才存在),而你又想要在本地进行验证你写得是否有问题，那么可以使用下面的命令。

```
// 将下面的变量替换成你自己的。 chart-name表示chart的名字,chart-dir表示chart地址
helm template --dry-run --debug --disable-openapi-validation ${chart-name} .\${chart-dir}\
```

然后如果你想要在k8s环境中安装的时候,而k8s环境又在远端服务器,那么你可以将chart打包，然后到服务器中进行安装，然后也可以在
将chart上传到服务器中，然后进行安装(服务器中要先安装helm)。


### 写到最后

自此,从springcloud迁移到k8s集群总算是完成了。因为是第一次使用helm(以前都用的kustomize),所以在helm这里耗费了一些功夫，主要是排查错误方面的,不过不得不说helm的文档写得不错，很清晰。

再然后就是代码改造以及一些配置问题,因为迁移azure,所以上面的关于它pipeline的一些配置不是很清楚，不过好在可以直接练习他们的运维,还是帮我们解决了一些问题的。

