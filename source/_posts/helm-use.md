---
title: helm常用命令
date: 2023-11-07 09:57:55
tags: [Helm, K8s]
---
### helm官方文档

hi, bro, 最好的永远是官方文档: https://helm.sh/zh/docs/

就是它上面的东西太多了，我知道你看得费力，所以我给你总结了下面的常用命令


### helm添加仓库
helm repo add  elastic  https://helm.elastic.co       
helm repo add  gitlab   https://charts.gitlab.io       
helm repo add  harbor   https://helm.goharbor.io
helm repo add traefik   https://traefik.github.io/charts    

//添加国内仓库       
helm repo add stable http://mirror.azure.cn/kubernetes/charts       
helm repo add aliyun https://kubernetes.oss-cn-hangzhou.aliyuncs.com/charts       
helm repo update       
helm repo list

<!--more-->


### 搜索chart

1. helm search repo traefik/traefik
2. helm search repo nginx

### 安装或者查看出错时

出现报错 Error: Kubernetes cluster unreachable: Get "http://localhost:8080/version": dial tcp 127.0.0.1:8080

需要指定kubernetes api server地址,可以通过

helm --kube-apiserver address


### 安装chart
1. helm repo update
2. helm install bitnami/mysql --generate-name
> 在 Helm 3 中，必须主动指定release名称，或者增加 --generate-name 的参数使Helm自动生成

### 查看安装的chart
1. helm list(helm ls)

### 卸载chart
1. helm uninstall mysql-xxx



### 安装traefik

1. 安装traefik,并指定对应参数
helm install --set deployment.kind=DaemonSet --set namespaceOverride=traefik --set service.enabled=false traefik traefik/traefik

2. 端口转发，像访问本地服务一样访问traefik
kubectl port-forward $(kubectl get pods --selector "app.kubernetes.io/name=traefik" -n traefik --output=name) 9000:9000 -n traefik

3. 更新服务
helm upgrade --set deployment.kind=DaemonSet --set namespaceOverride=traefik --set service.enabled=false traefik traefik/traefik

4. 配合Azure使用
helm install --set deployment.kind=DaemonSet --set namespaceOverride=traefik --set service.annotations."service.beta.kubernetes.io/azure-load-balancer-internal"=true  traefik traefik/traefik


### 安装时报错内存不足

0/2 nodes are available: 2 Insufficient memory. preemption: 0/2 nodes are available: 2 No preemption victims found for incoming pod

这是因为我们的chart在安装的时候，k8s集群内存不足。

可以通过下面的命令查看node的内存状况

```
kubectl describe node 
```
 
可以在安装的时候调整服务所需的内存,CPU等，也可以给集群加机器添加内存。


### 查看chart信息

+. 查看基本信息: helm show chart traefik/traefik
+. 查看所有信息: helm show all traefik/traefik

### 查看chart的可配置选项

1. helm show values traefik/traefik 

然后，你可以使用 YAML 格式的文件覆盖上述任意配置项，并在安装过程中使用该文件

2. echo '{web.port: 8080}' > values.yml
3. helm install -f values.yml traefik/traefik --generate-name

还可以直接使用 --set web.port=8080 指定,优先级比-f(--values)高


### 创建并安装自己的charts

1. helm create think-manifesto
2. heml package think-manifesto
3. helm install think-manifesto 上一步打包好的tgz包

### debug chart而不安装

helm install --debug --dry-run goodly-guppy ./mychart

这样不会安装应用(chart)到你的kubenetes集群中，只会渲染模板内容到控制台

如果想看渲染出错的内容,可以加上另外参数

helm install --dry-run --disable-openapi-validation moldy-jaguar ./mychart

### 调试

+ helm lint 是验证chart是否遵循最佳实践的首选工具。
+ helm template --debug 在本地测试渲染chart模板。
+ helm install --dry-run --debug：我们已经看到过这个技巧了，这是让服务器渲染模板的好方法，然后返回生成的清单文件。
+ helm get manifest: 这是查看安装在服务器上的模板的好方法。

helm template --dry-run --debug --disable-openapi-validation thinkpro-test .\think-manifesto\

### helm内置对象

https://helm.sh/zh/docs/chart_template_guide/builtin_objects/

#### Release

Release对象描述了版本发布本身。包含了以下对象：
1. Release.Name： release名称
2. Release.Namespace： 版本中包含的命名空间(如果manifest没有覆盖的话)

#### Values

Values对象是从values.yaml文件和用户提供的文件传进模板的。默认为空

#### Chart

Chart.yaml文件内容。 Chart.yaml里的所有数据在这里都可以可访问的。比如 {{ .Chart.Name }}-{{ .Chart.Version }} 会打印出 mychart-0.1.0

### 常用函数

模板中频繁使用的一个函数是default： default DEFAULT_VALUE GIVEN_VALUE。 这个函数允许你在模板中指定一个默认值，以防这个值被忽略. 当然还可以使用管道符配合使用,比如

```
drink: {{ .Values.favorite.drink | default "tea" | quote }}
```

### 删除空白

{{- (包括添加的横杠和空格)表示向左删除空白， 而 -}}表示右边的空格应该被去掉。 一定注意空格就是换行

要确保-和其他命令之间有一个空格。 {{- 3 }} 表示“删除左边空格并打印3”，而{{-3 }}表示“打印-3”。

### 模板的使用

我们一般会在_helpers.tpl中写我们的模板代码

#### 声明一个模板

```
{{- define "mychart.labels" }}
  labels:
    generator: helm
    date: {{ now | htmlDate }}
{{- end }}
```

#### 使用模板

我们可以使用 template  templateName 来引用我们的模板

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
  {{- template "mychart.labels" }}
```

如果我们在模板

```
{- define "mychart.labels" }}
  labels:
    generator: helm
    date: {{ now | htmlDate }}
    chart: {{ .Chart.Name }}
    version: {{ .Chart.Version }}
{{- end }}

```
中使用 .Chart,会发现渲染出错(--disable-openapi-validation 可以查看渲染的内容),这是因为没有内容传入到模板中(可以认为默认范围是在模板),所以无法使用 . 访问任何内容. 需要传递一个范围给模板

```
{{- template "mychart.labels" . }}
```
注意这个在template调用末尾传入的.，我们可以简单传入.Values或.Values.favorite或其他需要的范围。但一定要是顶层范围。

由于template是一个行为，不是方法，无法将 template调用的输出传给其他方法，数据只是简单地按行插入。

为了处理这个问题，Helm提供了一个 include 的可选项，可以将模板内容导入当前管道，然后传递给管道中的其他方法。

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-configmap
  labels:
{{ include "mychart.app" . | indent 4 }}
```

相较于使用template，在helm中使用include被认为是更好的方式 只是为了更好地处理YAML文档的输出格式


### 使用技巧

#### 配置更新后 Pod 自动重启

利用 k8s 的 Deployment 更改后的自动更新，我们可以用来更新应用配置，简单说就是更新 Secrets 或 ConfigMaps 后，计算它的最新 hash 值，然后将这个 hash 值 patch 到相应的 Deployment 中。

```yml
kind: Deployment
spec:
  template:
    metadata:
      annotations:
        checksum/config: {{ include (print $.Template.BasePath "/configmap.yaml") . | sha256sum }}
```

这样，假如这个配置有问题，比如造成应用崩溃了，k8s 也会认为新的 ReplicaSet 失败了，不会将流量导过去，从而不继续滚动更新，避免了了由配置更新导致的应用崩溃问题。

#### Image pull credentials
有些时候，Docker 镜像可能需要用户名与密码去 registry 拉取，那么，你就需要专门为此创建一个模板了。
比如 value 是：
```
imageCredentials:
  registry: quay.io
  username: someone
  password: sillyness
 ```

而模板就是：
```
{{- define "imagePullSecret" }}
{{- printf "{"auths": {"%s": {"auth": "%s"}}}" .Values.imageCredentials.registry (printf "%s:%s" .Values.imageCredentials.username .Values.imageCredentials.password | b64enc) | b64enc }}
{{- end }}
```
当然，需要注意多个 Deployment 共享一个 Chart 的情况，这时候可能会出现 secrets 冲突的情况，可考虑单为此单独创建一个 Config Chart，然后作为 App Chart 的依赖。

